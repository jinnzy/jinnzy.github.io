# 编译docker-compose arm版本


使用docker方式编译出来的文件在华为云鲲鹏上运行缺少库，所以选择直接在鲲鹏虚机上编译

# 安装python3

```bash
yum install git python36 python36-devel python36-pip zlib-devel -y

python3 -m pip install --upgrade pip  # 更新pip
```

# 开始编译

1. 拉取compose项目
```bash
git clone https://github.com/docker/compose.git
```
2. 创建`venv` 环境
```bash
cd compose/
python3 -m venv venv
source ./venv/bin/activate
python setup.py develop
```
3. 安装依赖
```bash
yum install libffi-devel 安装一些依赖库
python setup.py develop
pip install -r ./requirements-build.txt
```
4. 开始编译
```bash
pyinstaller -F ./docker-compose.spec
```

# 相关错误解决：

## ModuleNotFoundError: No module named 'setuptools_rust'

```bash
pip install --upgrade setuptools
```

## compose/GITSHA No such file or directory

删除以下内容，注意`,` 也要删除

```bash
    , 
			(
		        'compose/GITSHA',
		        'compose/GITSHA',
		        'DATA'
		    )
```

# 参考资料

[https://blog.csdn.net/qq_28808029/article/details/107221921](https://blog.csdn.net/qq_28808029/article/details/107221921)

