主库添加redo
相应的就会添加一组standby logifle

相应的就需要去备库操作
备库直接命令添加,会报错
需要先把实时恢复停掉,将standby_file_management设置为manual,方可正常添加
