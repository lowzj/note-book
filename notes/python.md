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

### READINGS
* [Python 编码转换与中文处理](http://www.jianshu.com/p/53bb448fe85b)

