<style>
  .container {
    position: relative;
  }

	.folder {
		padding: 0 0 0 1.5em;
		background: url(https://svelte.dev/tutorial/icons/folder.svg) 0 0.1em no-repeat;
		background-size: 1em 1em;
    color: rgb(55, 53, 47);
		font-weight: bold;
		cursor: pointer;
    user-select: none;
    display: block;
	}

	ul {
		padding: 0.2em 0 0 0.5em;
		margin: 0 0 0 0.5em;
		list-style: none;
		border-left: 1px solid #eee;
	}

	li {
		padding: 0.2em 0;
    transition: background 120ms ease-in 0s;
	}

  li:hover {
    background-color: rgba(55, 53, 47, 0.08);
  }

  li:active:hover {
    background-color: rgba(55, 53, 47, 0.16);
  }

  .addFileBtn {
    right: 0;
    top: 0;
    position: absolute;
    width: 12px;
    height: 12px;
    border-radius: 3px;
    border: 1px solid rgba(55, 53, 47, 0.16);
    display: flex;
    align-items: center;
    justify-content: center;
    user-select: none;
    transition: background 120ms ease-in 0s;
    cursor: pointer;
  }

  .addFileBtn:hover {
    background: rgba(55, 53, 47, 0.08);
  }

  .addFileBtn .plus {
    fill: rgba(55, 53, 47, 0.4);
  }

  .addFileBtn:hover .plus {
    fill: rgba(25, 23, 17, 0.6);
  }

  .addFileBtn:active:hover {
    background: rgba(55, 53, 47, 0.16);
  }

  .dropdown {
    position: absolute;
    top: 0px;
    right: 0;
    background: white;
    box-shadow: rgba(15, 15, 15, 0.05) 0px 0px 0px 1px, rgba(15, 15, 15, 0.1) 0px 3px 6px, rgba(15, 15, 15, 0.2) 0px 9px 24px;
    border-radius: 3px;
    padding-top: 6px;
    padding-bottom: 6px;
    z-index: 10;
  }

  .dropdown ul {
    border: none;
    padding: 0;
    margin: 0;
  }

  .dropdown li {
    padding-left: 0.6em;
    padding-right: 0.6em;
  }

  .dropdown div {
    color: rgba(25, 23, 17, 0.7);
    cursor: pointer;
    padding: 0 0 0 1.5em;
    background: 0 0.1em no-repeat;
    background-size: 1em 1em;
    font-weight: 500;
  }
</style>

<script>
  import { createEventDispatcher } from 'svelte';
	import File from './File.svelte';

	export let expanded = false;
  export let loaded=false;
	export let name;
	export let files;
  export let id;
  export let iconLink;
  export let visibleDropdown;

  let dropdownVisible = false;

	const dispatch = createEventDispatcher();

	function toggle() {
		expanded = !expanded;
    if (expanded) {
      dispatch('expand', {id});
    }
	}

  function handleContextMenu(e) {
    dispatch('contextmenu', {id, clientX: e.clientX, clientY: e.clientY});
  }

  function handleDropdown() {
    dispatch('dropdown', {id});
  }

  function handleCreate(type) {
    dispatch('create', {id, type});
  }

  const types = [
    {
      name: 'Doc',
      iconLink: 'https://drive-thirdparty.googleusercontent.com/16/type/application/vnd.google-apps.document',
      mimeType: 'application/vnd.google-apps.document',
    },
    {
      name: 'Sheet',
      iconLink: 'https://drive-thirdparty.googleusercontent.com/16/type/application/vnd.google-apps.spreadsheet',
      mimeType: 'application/vnd.google-apps.spreadsheet',
    },
    {
      name: 'Slide',
      iconLink: 'https://drive-thirdparty.googleusercontent.com/16/type/application/vnd.google-apps.presentation',
      mimeType: 'application/vnd.google-apps.presentation',
    },
    {
      name: 'Form',
      iconLink: 'https://drive-thirdparty.googleusercontent.com/16/type/application/vnd.google-apps.form',
      mimeType: 'application/vnd.google-apps.form',
    },
    {
      name: 'Folder',
      iconLink: 'https://drive-thirdparty.googleusercontent.com/16/type/application/vnd.google-apps.folder',
      mimeType: 'application/vnd.google-apps.folder',
    },
  ]
</script>

<div class='container' on:contextmenu|preventDefault={handleContextMenu}>
  <span class='folder' style='background-image: url({iconLink})' on:click={toggle}>{name}</span>
  <div class='addFileBtn' on:click|stopPropagation={handleDropdown}>
      <svg viewBox="0 0 18 18" class="plus" style="width: 10px; height: 10px; display: block; flex-shrink: 0; backface-visibility: hidden;"><polygon points="17,8 10,8 10,1 8,1 8,8 1,8 1,10 8,10 8,17 10,17 10,10 17,10 "></polygon></svg>
    <div style="position: relative">
      {#if visibleDropdown === id}
        <div class='dropdown'>
          <ul>
            {#each types as type}
              <li on:click={() => handleCreate(type)}>
                <div style="background-image: url({type.iconLink})">
                  {type.name}
                </div>
              </li>
            {/each}
          </ul>
        </div>
      {/if}
    </div>
  </div>
</div>

{#if expanded}
	<ul>
    {#if loaded}
      {#if files.length}
        {#each files as file}
          <li>
            {#if file.type === 'folder'}
              <svelte:self on:create on:expand on:dropdown on:contextmenu visibleDropdown={visibleDropdown} {...file}/>
            {:else}
              <File on:contextmenu {...file}/>
            {/if}
          </li>
        {/each}
      {:else}
        <li>Empty</li>
      {/if}
    {:else}
      <li>Loading</li>
    {/if}
	</ul>
{/if}
