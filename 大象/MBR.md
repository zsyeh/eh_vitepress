- 实模式下的内存布局

  | 开始  | 结束  | 大小  |                           用途                            |
  | :---: | :---: | :---: | :-------------------------------------------------------: |
  | FFFF0 | FFFFF |  16B  | BIOS入口，顶部640KB字节，只是为了强调入口地址才单独列出来 |
  |   0   | 9FFFF | 640kb |                           DRAM                            |

  

BIOS建立了中断向量表，，这样就可以通过int中断信号来实现相关硬件的调用

*BIOS实现这些功能都是对硬件IO的操作 所以名字也叫做 basic input output sys*

- 16位机器的地址总线为20位
- 32位机器地址总线为32位 其地址范围为4GB

值得注意的是，地址线所寻的地址并非全是物理内存

> 也就是说物理内存多大是没有用的，对于32位地址线的设备来说，内存超过了4GB就没有用了，因为剩下的一部分还需要用来访问外设的内存，所以计算机早期插入4G的内存只能够显示3.8G

总之，表示地址的那串数字是地址总线的输入，相当于其参数，和内存条无关，CPU能够访问这些地址，本质上是地址线的映射，CPU给地址总线提交一个地址

- bios上的ROM也是块内存， 被地址线映射在低端1MB内存的顶部，0xF0000--0xFFFFF
- bios也是一个程序，程序的入口地址**0xFFFF0**
- 在上电的一瞬间。CPU的cs:ip 被强制初始化0xF000：0xFFF0
- 执行开始的位置到1MB结束只有16字节所以真正的执行程序不在这个位置，跳转`jmp far f000:e05b`
- CHS&LBA
- magic number 0x7c00

挂起

- $ 就是标签地址

  ```assembly
  code_start:
  	mov ax,0
  	jump $;这个美元符号就是code_start:的地址
  ```

- `$$` 本section的起始地址

  如果该section使用vstart=xxxx修饰，`$$`的值就是xxxx 。$就是此起始地址的顺延

  如果使用vstart关键字，想要获取本section在文件中真实地址

  section.节名.start

  如果没有定义section,nasm默认全部代码属于一个section 起始地址0

  ```assembly
  section data
  	var dd 0
  section code
  	jump start
  ```

  ***值得注意的是，section是伪指令，CPU不关注***

  ```assembly
  SECTION MBR vstart=0x7c00
  	mov ax,cs    ;这个时候cs是0,使用通用寄存器ax进行中转，对于ds es ss fs这类sreg，cpu不可以直接赋值
  	             ;所以需要中转,没有立即数到段寄存器的物理电路实现
  	             
  	mov ds,ax
  	mov es,ax
  	mov ss,ax
  	mov fs,ax
  	mov sp,0x7c00   ;这是初始化stack pointer mbr也是程序,只要是程序就要用到stack,目前0x7c00后面暂时是安全
  	                ;区域,所以都当作栈来使用
  	
  	mov ax,0x600
  	mov bx,0x700
  	mov cx,0        ;left top corner(0,0)
  	mov dx,0x184f   ;right down corner(80,25)
  					;index start at 0 so:0x18=24 0x4f=79
                      ; in VGA mode a line contains 80 chars,25 lines in total
  	mov ah,0x06
  	int 0x10        ; interupt : clear       func num:0x06	
  	                ;因为前面可能有一些硬件自检的信息,所以需要先提前清楚掉
  	
  	
  	;;;;get the cursur position 
  	
  	mov ah,3       ;func num#3 :get_the_cursur_position()   0x10 是一个强大的中断，把功能号送到ah寄存器
  	mov bh,0       ;告诉寄存器是第零页
  	                ;显示器有很多模式 图形模式 文本模式
  	
  	int 0x10
  	;;;;;;;;;;;;;;;;;;;;;;;;;;
  	
  	mov ax,message
  	mov bp,ax
  	
  	
  	
  	mov cx,5
  	mov ax,0x1301 ;13号对应ah寄存器,01对应al寄存器
  	
  	mov bx,0x2
  	
  	int 0x10
  	
  	jmp $
  	
  	
  	message db "1 MBR"
  	times 510-($-$$) db 0 ;将空余部分填满,确保魔数在510-511  Define Byte 伪指令 填充字符
  	db 0x55,0xaa
  	
  ```

  