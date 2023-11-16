# Shallow clone by depth

[git clone](https://git-scm.com/docs/git-clone)


在 github action 中获取当前提交修改了哪些文件，但是一直读不到。

其实是 [actions/checkout@v3](https://github.com/actions/checkout) 时，fetch-depth 参数默认是 1，只 clone 了最近一次提交的内容。修改参数即可。

对应为 --depth 参数，可以设置为 0，表示 clone 所有的历史记录。

