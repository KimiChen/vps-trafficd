## 本目录工作要求
 - 本项目所有文件默认UTF-8格式

 ## Git 约定
 
 - 默认 git 是个公开 git
 - 用户已要求：以后每次改动后，按改动范围自己 先 `git pull`、`git add` 和 `git commit`，然后 `git push`，需要的 PrivateKey 路径 和 Passphrase 在 .env 文件中
 - 提交只包含本轮相关文件，不要把无关生成物混进去。
 - 生成物和依赖目录应保持 ignored，写入.gitignore
 - 提交信息明确，例如：
   - `修复：调整窗口标题栏布局`
   - `文档：添加代理指南`
   - `新增：更新工作区路径`
 - git pull、git push 的 PrivateKey 和 Passphrase 在 .env 文件中