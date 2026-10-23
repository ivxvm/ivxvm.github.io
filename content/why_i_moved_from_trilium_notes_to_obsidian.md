+++
title = "Why I moved from Trilium Notes to Obsidian"
date = 2026-08-23
+++

I've been using Trilium for a while and I really enjoyed it: 

* Hierarchic notes
* Rich text formatting
* Embedded media formats support
* Templates
* Properties
* Scripting and customization

However, after some time, some weak points also showed up:

* Database-backed, which means you won't see your note hierarchy as normal folders and files in filesystem, which has its pros and cons, but for me cons are overwhelming because it means I can't use regular tools, for example grep, or opening notes in other editors easily, or symlinking a folder from elsewhere as child items of the note
* No plugin infrastucture: even though Trilium is open-source, there's no built-in plugin support and rich plugin ecosystem like in Obsidian
* Even though rich text formatting is cool, it isn't great for compatibility, because it isn't always possible to convert this formatting to pure text formats: lossless export requires xml/html in general
* Theming isn't great, weak support to popular themes like Rose Pine, compared to Obsidian

Obsidian solves all those pain points for me. It also comes with its own set of issues though, and I'd like to address them:

* Built-in features are very lacking compared to Trilium: for example, folders by default can't be notes themselves, there's also no gallery view for folders of images, no custom sorting for notes, etc - all those issues are solvable with plugins thanks to the powerful plugin ecosystem, however it takes some time to set them up correctly
* Proprietary codebase: even though plugins are opensource, the Obsidian core app is proprietary, which is very unfortunate, but it is somewhat balanced out by its simple and open-ended architecture - the fact that everything in the vault is just regular folders and files makes it easy to create alternative frontends for the same vault format which don't even need to be associted to Obsidian itself in any way, and there are already examples of such projects: basalt (tui) and obsidian.nvim

Special mention needs to be made regarding embeds and backups:

* In Trilium I had to back up the whole vault, including any embedded images, atomically, as a single archive, my single backup size lately was exceeding 1GB, which is crazy for something that is supposed to be mostly about text, it happened because of images and media embeds ofc, but those in my case don't really belong to notes themselves and would benefit to be references to separate places in filesystem
* Because in Obsidian vault is just normal files, it becomes possible to bring images in by symlinking picture folders elsewhere in my filesystem, backup strategy also becomes more open-ended and even allows using VCS, and there are already plugins for git and such

Notable plugin mentions:

* Folder notes by Lost Paul - allows folders being notes themselves (actually there will be md file with the same name as folder either inside or outside of the corresponding folder depending on selected options, but Obsidian will show them as a single item in a smart way thanks to this plugin)
* Custom File Explorer sorting by SebastianMC - allows custom sorting of items in folders in multiple different ways, I use sortspec based approach which is very flexible and allows things like combining adhoc order for few items and then sorting the rest by metadata such as specified date property field in notes
* Gallery View by mkshp - allows using ` ```obs-gallery ...``` ` blocks in notes to render image galleries, I use it with image folders elsewhere in my filesystem symlinked into `vault/Symlinks/...`, I then use `path: "Symlinks/some_folder"` and it just works
* Iconize by Florian Woelki - allows adding icons to folders and notes
* Minimal Theme Settings by @kepano and Style Settings by mgmeyers - for nice and easy theme customizations
