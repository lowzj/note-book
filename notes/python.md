## Python

### shell 自动补全

```python
# python shell auto completer, add this to env:
#   export PYTHONSTARTUP=～/.startup.py
try:
  import readline
  import rlcompleter
  readline.parse_and_bind("tab: complete")
except:
  pass
```

### 创建虚拟环境

```sh
# create a virtual env
python -m venv venv
# enable virtual env
source ./venv/bin/activate
# check
which pip
```

---

### READINGS
* [Python 编码转换与中文处理](http://www.jianshu.com/p/53bb448fe85b)
* [pip 使用国内镜像源](https://www.runoob.com/w3cnote/pip-cn-mirror.html)
* [Python 中的依赖管理工具](https://www.kevinbai.com/articles/144.html)
* [Python七神器](https://zhuanlan.zhihu.com/p/355626817):
  * [Python Fire](https://github.com/google/python-fire)
  * [Python FastAPI](https://github.com/tiangolo/fastapi)

