# Construção de Compiladores - Daniel Lucrédio, Helena Caseli, Mário César San Felice e Murilo Naldi

### Demonstração 2 – Analisador sintático preditivo de descendência recursiva – ANTLR
---

1. Mostrar o site do ANTLR (www.antlr.org). Nesta demonstração faremos a instalação pelo Maven
2. Abrir o NetBeans e criar novo projeto Java Maven
- Project name: ```alguma-sintatico```
- Group Id: ```br.compiladores```
- Modificar o arquivo pom.xml para incluir a dependência para o ANTLR e o plugin do ANTLR

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>br.compiladores</groupId>
    <artifactId>alguma-sintatico</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>jar</packaging>
    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <maven.compiler.source>11</maven.compiler.source>
        <maven.compiler.target>11</maven.compiler.target>
    </properties>
    <build>
        <plugins>
            <plugin>
                <groupId>org.antlr</groupId>
                <artifactId>antlr4-maven-plugin</artifactId>
                <version>4.7.2</version>
                <executions>
                    <execution>
                        <id>antlr</id>
                        <goals>
                            <goal>antlr4</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>            
            <plugin>
                <artifactId>maven-assembly-plugin</artifactId>
                <configuration>
                    <archive>
                        <manifest>
                            <mainClass>br.compiladores.algumasintaticoantlr.Principal</mainClass>
                        </manifest>
                    </archive>
                    <descriptorRefs>
                        <descriptorRef>jar-with-dependencies</descriptorRef>
                    </descriptorRefs>
                </configuration>
                <executions>
                    <execution>
                        <id>make-assembly</id>
                        <phase>package</phase>
                        <goals>
                            <goal>single</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
    <dependencies>
        <dependency>
            <groupId>org.antlr</groupId>
            <artifactId>antlr4</artifactId>
            <version>4.7.2</version>
            <classifier>complete</classifier>
        </dependency>
    </dependencies>    
</project>
```

3. Criar um novo arquivo do tipo ANTLR Grammar, chamado ```Alguma.g4```

- O plugin ANTLR-Maven exige que o arquivo com a gramática seja incluído em um diretório específico dentro da pasta src/main do projeto. Esse diretório só pode ser criado na aba “arquivos/files” do NetBeans. Esse diretório deve se chamar “antlr4” e deve ter a mesma estrutura de diretórios que os pacotes Java. Observe que é necessário alternar para a aba “arquivos/files”.

- Ao retornar para a aba “projetos/projects” a nova estrutura de diretórios irá aparecer dentro de “other sources”
- O conteúdo do arquivo é o seguinte

```
grammar Alguma;

TIPO_VAR 
	:	'INTEIRO' | 'REAL';

NUMINT
	:	('0'..'9')+
	;

NUMREAL
	:	('0'..'9')+ ('.' ('0'..'9')+)?
	;
	
VARIAVEL
	:	('a'..'z'|'A'..'Z') ('a'..'z'|'A'..'Z'|'0'..'9')*
	;

CADEIA
	:	'\'' ( ESC_SEQ | ~('\''|'\\') )* '\''
	;
	
OP_ARIT1
	:	'+' | '-'
	;

OP_ARIT2
	:	'*' | '/'
	;

OP_REL
	:	'>' | '>=' | '<' | '<=' | '<>' | '='
	;

OP_BOOL	
	:	'E' | 'OU'
	;
	
fragment
ESC_SEQ
	:	'\\\'';

COMENTARIO
	:	'%' ~('\n'|'\r')* '\r'? '\n' {skip();}
	;

WS 	:	( ' ' |'\t' | '\r' | '\n') {skip();}
	;
	
programa
	:	':' 'DECLARACOES' listaDeclaracoes ':' 'ALGORITMO' listaComandos
	;
	
listaDeclaracoes
	:	declaracao listaDeclaracoes | declaracao
	;
	
declaracao
	:	VARIAVEL ':' TIPO_VAR
	;
	
expressaoAritmetica
	:	expressaoAritmetica OP_ARIT1 termoAritmetico
	|	termoAritmetico
	;
	
termoAritmetico
	:	termoAritmetico OP_ARIT2 fatorAritmetico
	|	fatorAritmetico
	;
	
fatorAritmetico
	:	NUMINT
	|	NUMREAL
	|	VARIAVEL
	|	'(' expressaoAritmetica ')'
	;
	
expressaoRelacional
	:	expressaoRelacional OP_BOOL termoRelacional
	|	termoRelacional
	;
	
termoRelacional
	:	expressaoAritmetica OP_REL expressaoAritmetica
	|	'(' expressaoRelacional ')'
	;
	

listaComandos
	:	comando listaComandos
	|	comando
	;
	
comando
	:	comandoAtribuicao
	|	comandoEntrada
	|	comandoSaida
	|	comandoCondicao
	|	comandoRepeticao
	|	subAlgoritmo
	;
	
comandoAtribuicao
	:	'ATRIBUIR' expressaoAritmetica 'A' VARIAVEL
	;
	
comandoEntrada
	:	'LER' VARIAVEL
	;
comandoSaida
	:	'IMPRIMIR' (VARIAVEL | CADEIA)
	;
	
comandoCondicao
	:	'SE' expressaoRelacional 'ENTAO' comando
	|	'SE' expressaoRelacional 'ENTAO' comando 'SENAO' comando
	;
	
comandoRepeticao
	:	'ENQUANTO' expressaoRelacional comando
	;
subAlgoritmo
	: 'INICIO' listaComandos 'FIM'
	;
```

4. Mandar gerar o reconhecedor
- Basta clicar com o botão direito no projeto e selecionar a opção “Build/Construir”. Será gerada uma nova pasta de código-fonte, chamada “Generated sources (antlr4)”, onde o código gerado terá a estrutura correta de pacotes.
- Caso exista algum erro no arquivo da gramática, o processo irá gerar um erro. Observar na janela de “Saída/output” para identificar a origem do erro.
5. Criar a classe Principal:

```java
package br.compiladores.alguma.sintatico;

import java.io.IOException;
import org.antlr.v4.runtime.CharStream;
import org.antlr.v4.runtime.CharStreams;
import org.antlr.v4.runtime.CommonTokenStream;
import org.antlr.v4.runtime.Token;

public class Principal {
    public static void main(String args[]) throws IOException {
        CharStream cs = CharStreams.fromFileName(args[0]);
        AlgumaLexer lexer = new AlgumaLexer(cs);

//        // Descomentar para depurar o Léxico
//        Token t = null;
//        while( (t = lexer.nextToken()).getType() != Token.EOF) {
//            System.out.println("<" + AlgumaLexer.VOCABULARY.getDisplayName(t.getType()) + "," + t.getText() + ">");
//        }
        CommonTokenStream tokens = new CommonTokenStream(lexer);
        AlgumaParser parser = new AlgumaParser(tokens);
        parser.programa();
    }
}
```

6. Executar (vai dar erro na expressão booleana)
- Construir o projeto novamente. Devido às configurações do arquivo pom.xml, será gerado na pasta “target” um arquivo .jar com todas as dependências necessárias para execução (incluindo o runtime do antlr)
- Para executar, abrir um terminal e executar o comando (tudo em uma linha só, não esquecer de substituir os caminhos deste exemplo pelos reais)

```sh
java -jar /Users/NetBeansProjects/alguma-sintatico/target/alguma-sintatico-1.0-SNAPSHOT-jar-with-dependencies.jar /Users/Desktop/teste.txt
```

7. Fazer a depuração do léxico (descomentar as linhas acima)
- Mostrar que está interpretando "E" e "OU" como variáveis
8. Mover a regra OP_BOOL para antes da regra VARIAVEL
9. Refazer os testes (é preciso comentar novamente as linhas acima)
10. Inserir alguns erros e testar
- Mostrar que alguns erros entre comandos ele não detecta, ex:

```
:ALGORITMO
% Coloca 3 números em ordem crescente
LER numero1
LER numero2
a b c d
LER numero3
SE numero1 > numero2 ENTAO
```
- Na verdade, está encerrando a análise prematuramente. Para obrigar que um programa esteja 100% correto, é preciso inserir EOF no final da primeira regra:

```
Programa: ':' 'DECLARACOES' listaDeclaracoes ':' 'ALGORITMO' listaComandos EOF;
```

- Mostrar o código gerado para a regra listaComandos, destacando a recursividade. Trocar pela regra (comando)+ e mostrar que a implementação é diferente.
- Fazer o mesmo para lista de declarações
11. Inserir algumas ações para imprimir o que está reconhecendo

```
programa
	:	 { System.out.println("Começou um programa"); }
		':' 'DECLARACOES' 
		{ System.out.println("  Declarações"); }
		listaDeclaracoes ':' 'ALGORITMO' 
		{ System.out.println("  Algoritmo"); }
		listaComandos
	;
declaracao
	:	VARIAVEL ':' TIPO_VAR
		{ System.out.println("    Declaração: Var="+$VARIAVEL.text+", Tipo="+$TIPO_VAR.text); }
	;
comandoAtribuicao
	:	'ATRIBUIR' expressaoAritmetica 'A' VARIAVEL
		{ System.out.println("      "+$VARIAVEL.text+" = "+$expressaoAritmetica.text); }
	;
comandoEntrada
	:	'LER' VARIAVEL
		{ System.out.println("      "+$VARIAVEL.text+" = ENTRADA"); }
	;
comandoSaida
	:	'IMPRIMIR' texto=(VARIAVEL| CADEIA)
		{ System.out.println("      IMPRIMIR "+$texto.text); }
	;
```
12. Testar
13. Mostrar o uso de retorno

```
listaComandos : cmd=comando { System.out.println("Apareceu um comando do tipo "+$cmd.tipoComando); } listaComandos
	|	cmd=comando { System.out.println("Apareceu um comando do tipo "+$cmd.tipoComando); };
	
comando returns [ String tipoComando ]
	:	comandoAtribuicao { $tipoComando = "Atribuicao"; }
	|	comandoEntrada { $tipoComando = "Entrada"; }
	|	comandoSaida { $tipoComando = "Saida"; }
	|	comandoCondicao { $tipoComando = "Condicao"; }
	|	comandoRepeticao { $tipoComando = "Repeticao"; }
	|	subAlgoritmo { $tipoComando = "Subalgoritmo"; };
```
14. Testar
15. Mostrar outro tipo de retorno

```
programa : ':' 'DECLARACOES' listaDeclaracoes ':' 'ALGORITMO' lc=listaComandos { System.out.println("Numero de comandos: "+$lc.numComandos); };

listaComandos returns [ int numComandos ] : cmd=comando { System.out.println("Apareceu um comando do tipo "+$cmd.tipoComando); } lc=listaComandos { $numComandos = $lc.numComandos + 1;}
	|	cmd=comando { System.out.println("Apareceu um comando do tipo "+$cmd.tipoComando); $numComandos = 1;};
```
16. Testar


