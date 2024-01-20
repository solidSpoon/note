# Windows 交换 ESC 和 CapsLock

将下面代码保存成 `.reg` 文件

```
Windows Registry Editor Version 5.00

[HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Keyboard Layout]
"Scancode Map"=hex:00,00,00,00,00,00,00,00,03,00,00,00,3a,00,01,00,01,00,3a,00,00,00,00,00
```

运行然后重启

# 恢复

<procedure title="恢复" type="steps" id="">
    <step>Win + R</step>
    <step>运行 `regedit`</step>
    <step>打开 `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Keyboard Layout`</step>
    <step>删除该映射文件，再重启电脑，键盘就可以恢复按键原本的位置了。</step>
</procedure>