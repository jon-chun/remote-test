Typo — you typed `fix-text` but your branch is `fix-test`. Fix:

```bash
git push origin fix-test
```

Since this branch doesn't have an upstream yet, you'll probably want:

```bash
git push -u origin fix-test
```

so future `git push`/`git pull` on this branch don't need the branch name.

One more thing worth a look: your commit added `.DS_Store` (a macOS Finder metadata file, not something you want in the repo). If you'd like it gone:

```bash
git rm --cached .DS_Store
echo ".DS_Store" >> .gitignore
git add .gitignore
git commit -m "Remove .DS_Store, add to gitignore"
```