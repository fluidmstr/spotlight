# Publishing this wiki

GitHub creates the wiki git repository only after the first page exists in the web UI.

1. Open https://github.com/fluidmstr/spotlight/wiki and click **Create the first page**.
2. Save any placeholder (for example, the content of Home.md).
3. Then from this folder:

   git init -b master
   git add .
   git commit -m "docs: initialize wiki"
   git remote add origin https://github.com/fluidmstr/spotlight.wiki.git
   git push -u origin master --force

After the first push, edit pages here and push again. The files in docs/ remain
the source of truth; copy them over before publishing.