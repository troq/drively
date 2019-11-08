Drively - File explorer for Google Drive
=============================

[![Build Status](https://travis-ci.org/philc/vimium.svg?branch=master)](https://travis-ci.org/philc/vimium)

Drively is a file explorer for Google Drive written in [Svelte](https://svelte.dev). Here are the features of Drively:

- Browse and open files in one click
- Quickly create Docs, Sheets, Slides, Forms, and Folders in any location (My Drive, subfolders, etc)
- Right click any file/folder to copy its address or delete it

*NOTE* Drively uses [chrome.identity](https://developer.chrome.com/apps/identity) to access your Google Drive. It never talks to any servers and does absolutely nothing with your data. Read the source code if you're curious.

__Installation instructions:__

1. Clone the repository.
2. Install [yarn](https://yarnpkg.com/lang/en/docs/install/).
3. Run `yarn`.
4. Run `npm run start`
5. Load your extension on Chrome following:
    1. Access `chrome://extensions/`
    2. Check `Developer mode`
    3. Click on `Load unpacked extension`
    4. Select the `build` folder.
6. Have fun.

License
-------
Copyright (c) Tieshun Roquerre. See MIT-LICENSE.txt for details.
