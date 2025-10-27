# The Hitchhiker's Lexicon

A personal, multi-language repository serving as both a digital toolkit and a knowledge base. This is where small programs, useful automation scripts, well-defined code snippets, and general programming notes are collected, refined, and organized for quick reference and reuse.

The focus is on practical utility and deep understanding (Lexicon), gathered from years of coding experience (Hitchhiker).



## Sparse Checkout

Clone the repo, but make it empty.

```
git clone --filter=blob:none --no-checkout <repository-url> your-local-repo
cd your-local-repo
```

Enable sparse checkout

```
git sparse-checkout init
```

Specify the directories to keep in local repo

```
# Only bring down the 'scripts/test' directory and the README file
git sparse-checkout set scripts/test README.md
```

Use git as normal.



