<script>
  import { onMount } from 'svelte';
	import Folder from './Folder.svelte';
  import util from 'util';
  import copy from './copy.js';

  let visibleDropdown;
  let menuX, menuY, menuId, menuVisible = false;
  let root = {id: 'root', files: [], loaded: false};

  function openContextMenu(e) {
    if (e.detail.id === root.id) return;
    menuX = e.detail.clientX;
    menuY = e.detail.clientY;
    menuId = e.detail.id;
    menuVisible=true;
    visibleDropdown = undefined;
  }

  function openDropdown(e) {
    visibleDropdown = visibleDropdown === e.detail.id? undefined : e.detail.id;
    menuVisible = false;
  }

  function handleClick(e) {
    menuVisible = false;
    visibleDropdown = undefined;
  }

  const oauthPromise = new Promise((resolve, reject) => {
    chrome.identity.getAuthToken({interactive: true}, resolve);
  });


  function isFolder(file) {
    return file.mimeType === 'application/vnd.google-apps.folder';
  }

  function isParent(file, parent) {
    if (parent === undefined && file.parents === undefined) {
      return true;
    } else if (parent !== undefined && file.parents !== undefined) {
      for (let i = 0; i < file.parents.length; i++) {
        if (file.parents[i] === parent) {
          return true;
        }
      }
    }

    return false;
  }

  function buildTree(files, parent) {
    const tree = [];

    files.forEach(f => {
      if (isParent(f, parent)) {
        let node = {
          id: f.id,
          type: 'file',
          name: f.name,
          webViewLink: f.webViewLink,
          iconLink: f.iconLink,
          mimeType: f.mimeType,
          parents: f.parents,
        };

        if (isFolder(f)) {
          node.type = 'folder';
          node.files = buildTree(files, f.id);
        }

        tree.push(node);
      }
    });

    return tree;
  }

  function getFilesInFolder(parent) {
    return getFilesInFolders([parent]);
  }

  async function getFilesInFolders(parents) {
    const token = await oauthPromise;
    const init = {
      method: 'GET',
      async: true,
      headers: {
        Authorization: 'Bearer ' + token,
        'Content-Type': 'application/json'
      },
      'contentType': 'json'
    };
    return (await fetch(
      `https://www.googleapis.com/drive/v3/files?q=(${parents.map(parent => `"${parent}"%20in%20parents`).join('%20or%20')})%20and%20trashed%3Dfalse&pageSize=1000&fields=files(id%2CmimeType%2Cparents%2CiconLink%2CwebViewLink%2Cname)`,
      init)).json();
  }

  function getLocalRoot() {
    return new Promise((resolve, reject) => {
      chrome.storage.local.get(['root'], function(result) {
        resolve(result.root);
      });
    });
  }

  let y;
  function getLastY() {
    return new Promise((resolve, reject) => {
      chrome.storage.local.get(['y'], function(result) {
        resolve(result.y);
      });
    });
  }

  async function removeCachedAuthToken() {
    const token = await oauthPromise;
    return new Promise((resolve, reject) => chrome.identity.removeCachedAuthToken({token}, resolve));
  }

  async function updateRoot(currentRoot, files) {
    const id = files[0].parents[0];
    const r = {id, files: buildTree(files, id), loaded: true};
    const loaded = listLoaded(currentRoot).filter(fid => fid !== id);
    const resp = await getFilesInFolders(loaded);
    if (resp.error) {
      alert('Google Drive API request failed. Please msg tieshun.roquerre@gmail.com to debug.');
      return r;
    }
    const topLevel = loaded.filter(fid => findFile(fid, r));
    const notTopLevel = loaded.filter(fid => !findFile(fid, r));
    topLevel.forEach(fid => {
      const folder = findFile(fid, r);
      folder.files = buildTree(resp.files, fid);
      folder.loaded = true;
      folder.expanded = findFile(fid, currentRoot).expanded;
    });
    notTopLevel.forEach(fid => {
      const folder = findFile(fid, r);
      if (folder) {
        folder.loaded = true;
        folder.expanded = findFile(fid, currentRoot).expanded;
      }
    });
    return r;
  }

  function listFolders(root) {
    let folders = [root.id];
    root.files.filter(f => f.files).forEach(f => folders = folders.concat(listFolders(f)));
    return folders;
  }

  function listLoaded(root) {
    let folders = [root.id];
    root.files.filter(f => f.loaded).forEach(f => folders = folders.concat(listLoaded(f)));
    return folders;
  }

  async function getRoot(error) {
    root = (await getLocalRoot()) || root;
    const resp = await getFilesInFolder('root');
    if (resp.error) {
      if (error) {
        alert('Could not complete the Google OAuth process. Please msg tieshun.roquerre@gmail.com to debug.');
        return;
      }
      await removeCachedAuthToken();
      return getRoot(resp.error);
    }
    root = await updateRoot(root, resp.files);
  }

  function findFile(id, r) {
    r = r || root;
    let stack = [];
    stack.push(r);

    while (stack.length > 0) {
        let node = stack.pop();
        if (node.id === id) {
            return node;
        } else if (node.files && node.files.length) {
            for (let i = 0; i < node.files.length; i++) {
                stack.push(node.files[i]);
            }
        }
    }

    return null;
  }

  async function toggleFolder(e) {
    const folder = findFile(e.detail.id);
    if (!folder) return;
    folder.expanded = e.detail.expanded;
    if (!folder.loaded) {
      const resp = await getFilesInFolder(folder.id);
      folder.files = buildTree(resp.files, folder.id);
      folder.loaded = true;
    }
    root = root;
  }

  async function deleteFile(id) {
    const file = findFile(id);
    const parent = findFile(file.parents[0]);

    const oldFiles = parent.files;
    parent.files = parent.files.filter(f => f.id !== id);
    menuVisible = false;
    root = root;

    const token = await oauthPromise;
    const init = {
      method: 'PATCH',
      async: true,
      headers: {
        Authorization: 'Bearer ' + token,
        'Content-Type': 'application/json'
      },
      contentType: 'json',
      body: JSON.stringify({ trashed: true })
    };
    const resp = await fetch(
      `https://www.googleapis.com/drive/v3/files/${id}`,
      init);
    if (!resp.ok) {
      alert(`Could not trash file ${file.name} with id ${id}`);
      parent.files = oldFiles;
      root = root;
    }
  }

  async function createFile(e) {
    const folder = findFile(e.detail.id);
    const file = {
      id: null,
      type: e.detail.type.mimeType === 'application/vnd.google-apps.folder' ? 'folder' : 'file',
      name: 'Creating...',
      webViewLink: '#',
      iconLink: e.detail.type.iconLink,
      mimeType: e.detail.type.mimeType,
      parents: [e.detail.id],
    }
    folder.files.unshift(file);
    root = root;
    const token = await oauthPromise;
    const init = {
      method: 'POST',
      async: true,
      headers: {
        Authorization: 'Bearer ' + token,
        'Content-Type': 'application/json'
      },
      contentType: 'json',
      body: JSON.stringify({
        parents: [e.detail.id],
        mimeType: e.detail.type.mimeType,
      })
    };
    const resp = await fetch(
      'https://www.googleapis.com/drive/v3/files?fields=id%2Cname%2CwebViewLink%2CiconLink',
      init);
    if (resp.ok) {
      const data = await resp.json();
      Object.assign(file, data);
    } else {
      alert(`Could not create ${e.detail.type.name}`);
      folder.files.shift();
    }
    root = root;
  }

  $: menuFile = findFile(menuId);

  getRoot();
  onMount(async () => {
    window.scrollTo(0, await getLastY());
	});
  $: chrome.storage.local.set({root}, function() {
    console.log('Updated root to:\n' + util.inspect(root, {showHidden: false, depth: null}));
  });
  $: chrome.storage.local.set({y}, function() {
    console.log(`Updated y to: ${y}`);
  });
</script>

<style>
  .container {
    min-width: 300px;
    background-color: rgb(247, 246, 243);
    padding: 8px;
  }

  .menu {
    background: white;
    border-radius: 3px;
    box-shadow: rgba(15, 15, 15, 0.05) 0px 0px 0px 1px, rgba(15, 15, 15, 0.1) 0px 3px 6px, rgba(15, 15, 15, 0.2) 0px 9px 24px;
    position: fixed;
    padding-top: 6px;
    padding-bottom: 6px;
    min-width: 80px;
  }

  .menu div {
    color: rgb(55, 53, 47);
    transition: background 120ms ease-in 0s;
    user-select: none;
    display: flex;
    justify-content: start;
    align-items: center;
    cursor: pointer;
    padding: 0.3em 0.6em;
  }

  .menu div:hover {
    background: rgba(55, 53, 47, 0.08);
  }

  .menu div:active:hover {
    background: rgba(55, 53, 47, 0.16);
  }

  .menu svg {
    margin-left: 4px;
  }

  .menu span {
    margin-left: 4px;
    margin-right: 4px;
  }
</style>

<svelte:window bind:scrollY={y} on:click={handleClick}/>

<div class='container'>
  <Folder {...root} iconLink='https://drive-thirdparty.googleusercontent.com/16/type/application/vnd.google-apps.folder+shared' name="My Drive" visibleDropdown={visibleDropdown} expanded on:toggle={toggleFolder} on:create={createFile} on:dropdown={openDropdown} on:contextmenu={openContextMenu}/>
</div>

{#if menuVisible}
  <div class='menu' class:menuVisible on:click={e => e.stopPropagation()} style="top: {menuY}px; left: {menuX}px">
    <div on:click={() => deleteFile(menuId)}>
      <svg viewBox="0 0 30 30" class="trash" style="width: 12px; height: 12px; display: block; fill: inherit; flex-shrink: 0; backface-visibility: hidden;"><path d="M21,5c0-2.2-1.8-4-4-4h-4c-2.2,0-4,1.8-4,4H2v2h2v22h22V7h2V5H21z M13,3h4c1.104,0,2,0.897,2,2h-8C11,3.897,11.897,3,13,3zM24,27H6V7h18V27z M16,11h-2v12h2V11z M20,11h-2v12h2V11z M12,11h-2v12h2V11z"></path></svg>
      <span>Trash {menuFile.type} {menuFile.name}</span>
    </div>
    <div on:click={() => {copy(menuFile.webViewLink); menuVisible = false;}}>
      <svg viewBox="0 0 30 30" class="link" style="width: 17px; height: 17px; display: block; fill: inherit; flex-shrink: 0; backface-visibility: hidden;"><path d="M2,12c0-3.309,2.691-6,6-6h8c3.309,0,6,2.691,6,6s-2.691,6-6,6h-6c0,0.736,0.223,1.41,0.574,2H16c4.418,0,8-3.582,8-8 c0-4.418-3.582-8-8-8H8c-4.418,0-8,3.582-8,8c0,2.98,1.634,5.575,4.051,6.951C4.021,18.638,4,18.321,4,18 c0-0.488,0.046-0.967,0.115-1.436C2.823,15.462,2,13.827,2,12z M25.953,11.051C25.984,11.363,26,11.68,26,12 c0,0.489-0.047,0.965-0.117,1.434C27.176,14.536,28,16.172,28,18c0,3.309-2.691,6-6,6h-8c-3.309,0-6-2.691-6-6s2.691-6,6-6h6 c0-0.731-0.199-1.413-0.545-2H14c-4.418,0-8,3.582-8,8c0,4.418,3.582,8,8,8h8c4.418,0,8-3.582,8-8 C30,15.021,28.368,12.428,25.953,11.051z"></path></svg>
      <span>Copy link</span>
    </div>
  </div>
{/if}

