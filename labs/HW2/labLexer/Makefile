CC       = gcc -g
LEX      = flex -i -I 
INC      = -I../pl0/include 

all: rellex rel

rel: src/labLexer-2.c
	$(CC) $(INC) src/labLexer-2.c -o labLexer-2m

rellex: grammar/rel.lex
	cd grammar; $(LEX) rel.lex
	$(CC) $(INC) grammar/lex.yy.c -DLEXERGEN src/labLexer-2.c -o labLexer-2 -ll

clean:
	rm -f *.BAK *.o core *~* *.a 
	rm -f *.tab.h *.tab.c
	rm -f expr *.out
