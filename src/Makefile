## Makefile

TARGET		= galasm
OBJS		= galasm.o support.o jedec.o localize.o
CC          = gcc

CFLAGS		= -Wall -O2
LDFLAGS		=


all: $(TARGET)

$(TARGET): $(OBJS)
	gcc $(LDFLAGS) -o $@ $(OBJS)

check: $(TARGET)
	python3 ../tests/run_tests.py --legacy --galasm ./$(TARGET)

clean:
	-rm $(OBJS)
	-rm -f *.[oa] $(TARGET) Makefile.bak

cleantilde:
	/usr/bin/find . -regex ".+~$$" -exec /bin/rm {} \;
 
