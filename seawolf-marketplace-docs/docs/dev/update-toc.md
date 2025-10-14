# Add a Menu Item

Menu items are added by updateing the code under the nav section of the `mkdocs.yml` file.
Each first column of dashes is a top level menu item and if that menu item is just a container to hold other pages (as with this section `DEV TEAM LOOK HERE TO ADD YOUR DOCS`), you end the line with just a colon. If it represents a page, itself, you end with a colon and then the file name relative to the docs folder.

For example,

```text
/docs/
├── assets
│   ├── 1200px-Alaska_Anchorage_Seawolves_logo.svg.png
│   ├── favicon.ico
│   └── uaa-seawolves-favicon.png
├── dev
│   ├── add-docs.md
│   ├── clone.md
│   ├── file-location.md
│   └── update-toc.md
├── getting-started.md
├── index.md
├── stylesheets
│   └── uaa.css
└── technical-docs
    └── server-info.md
```

The file structure above is represented in the mkdocs.yml by the following nav block:

```yml
nav:
  - Home: index.md
  - DEV TEAM LOOK HERE TO ADD YOUR DOCS:
    - 1. Clone Docs: dev/clone.md
    - 2. How to Add Docs:
      - File Creation Location: dev/file-location.md
      - Update TOC: dev/update-toc.md
      - Add Documentation: dev/add-docs.md
  - Getting Started: getting-started.md
  - Features: features.md
  - API Reference: api.md
  - FAQ: faq.md
  - Technical Docs:
    - Server Info: technical-docs/server-info.md
```

Finally, [add your docmentation](add-docs.md)