# 解决Windows中&quot;文件正在使用&quot;问题

当我们试图修改或删除某个文件时，可能收到一个“文件正在使用”的提示。

## 通过资源监视器找到使用者

<procedure title="打开资源监视器" type="choices" id="open-resource-manager">
   <step>使用快捷键 <shortcut>Win+R</shortcut> 打开"运行"对话框，输入 resmon.exe，按回车键运行。</step>
   <step>点击开始菜单，前往<ui-path> 所有程序 | Windows 管理工具 | 资源监视器</ui-path>。</step>
</procedure>

<procedure title="使用资源监视器关闭目标程序" type="steps" id="procedure-id">
   <step>选择 "CPU" 标签</step>
   <step>在“关联的句柄”搜索框中输入您想要释放的文件名</step>
   <step>右键点击相应的进程，选择 "结束进程"</step>
</procedure>