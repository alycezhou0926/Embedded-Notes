# Git上传指令

### 更新

```c
git add .
git commit -m "这里写今天干了啥"
git push
```

### 第一次提交
```
# 修改默认主分支（master -> main）
git branch -M main
```

```c
# 1. 重新初始化
git init

# 2. 装箱 (注意add后面有个空格和点)
git add .

# 3. 封箱 (这次我们写一个标准的初始化备注)
git commit -m "Initial commit: 嵌入式笔记架构初始化"

# 4. 贴快递单 (换成你新建的仓库地址)
git remote add origin https://github.com/alycezhou0926/Embedded-Notes.git

# 5. 发货 (因为是第一次，必须加 -u)
git push -u origin main
```

