Makefile
========

注意事项
--------

1. **VPATH**\ 这个变量很重要

.. code:: makefile

   CC ?= gcc

   SRCDIRS	:= driver lib .
   INCDIRS	:=driver include lib 

   SRCFILES	:= $(foreach dir, $(SRCDIRS), $(wildcard $(dir)/*.c))
   CFILES	:=$(notdir $(SRCFILES))

   INCLUDES	:=$(patsubst %, -I %, $(INCDIRS))
   OBJFILES	:= $(patsubst %.o, obj/%.o, $(CFILES:.c=.o))

   .PHONY: clean 
   TARGET	:=main
   VPATH :=$(SRCDIRS)   #有了这个 make才会去搜索源文件 否则【1】处将找不到源文件报错

   $(TARGET):$(OBJFILES)
   	$(CC) -o  $@ $^

   $(OBJFILES):obj/%.o:%.c  #【1】要在链接的时候使用模式变量就一定要使用%，不然$<将会始终匹配到第一个依赖文件
   	$(CC) -c $(INCLUDES) -o $@ $<


   clean:
   	rm -rf obj/*.o

文件结构
--------

   project

   -->driver

   -->test1.c

   -->test1.h

   -->include

   -->inc.h

   -->lib

   -->lib.c

   -->lib.h

   ---main.c
