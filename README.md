# marp-test

## Presentaion tool set

Useful tool set to make a slide show for a presentation,
with marp (markdown presentation builder) and mermaid (graph builder by text)

## How to use

This tool set requires node.js

Run:

```
npm install @marp-team/marp-cli
npm install @mermaid-js/mermaid-cli
```

to install marp and mermaid (with option -g, globally install).

Then run:

```
npm run dev
```

and then will build a slide (automatically build svg images from mmd files).

Access `localhost:8080` with your browser to see the slide show.

## Package

As you can see in `package.json`, use the following package of node.js:

    "@marp-team/marp-cli": "^4.0.4"

    "@mermaid-js/mermaid-cli": "^11.4.2"
