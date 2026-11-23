---
created: 2026-08-12T10:17
---


---
> 以后所有优质的 skill 统一放在 my-skill 文件夹下，依照现有的 skill 模板，可以先积累别人的优秀 skill，后续慢慢结合优质 skill 以及实际业务积累写自己的定制 skill

### 初始化

增加 skill 的时候在对应的目录执行跟本地 CLI 建立软连接「Claude CodeX Cursor」的脚本

	./install-all.sh

卸载 skill 的脚本

	./uninstall-all.sh


---

### Agent 执行规范

- 增加description的区分度，利于 Agent 调用
	
- 显示触发： `Use Skill: XXXX`

想要提高自动执行精确度一定要提高description 质量



