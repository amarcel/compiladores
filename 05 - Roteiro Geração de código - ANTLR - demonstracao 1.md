# Construção de Compiladores - Daniel Lucrédio, Helena Caseli, Mário César San Felice e Murilo Naldi

### Demonstração 1 – Gerador de código C
---

1. Criar um arquivo de teste no Desktop

```
:DECLARACOES
num:INTEIRO
potencia:INTEIRO
aux:INTEIRO
resultado:INTEIRO
:ALGORITMO
LER num
LER potencia
SE potencia = 0
ENTAO
   ATRIBUIR 1 A resultado
SENAO
   INICIO
      ATRIBUIR num A resultado
      ATRIBUIR 1 A aux
      ENQUANTO aux < potencia
         INICIO
            ATRIBUIR resultado * num A resultado
            ATRIBUIR aux + 1 A aux
         FIM
   FIM
IMPRIMIR 'O resultado é:'
IMPRIMIR resultado
```

2. Criar um novo projeto Java Maven no NetBeans
- Project name: ```alguma-gerador```
- Group Id: ```br.compiladores.compiladores```
- Modificar o arquivo pom.xml para incluir a dependência para o ANTLR e o plugin do ANTLR

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" 
...
    <build>
        <plugins>
            <plugin>
                <groupId>org.antlr</groupId>
                <artifactId>antlr4-maven-plugin</artifactId>
                <version>4.7.2</version>
                <configuration>
                    <visitor>true</visitor>
                </configuration>
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
<mainClass>br.compiladores.compiladores.alguma.gerador.Principal</mainClass>
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

- O plugin ANTLR-Maven exige que o arquivo com a gramática seja incluído em um diretório específico dentro da pasta src/main do projeto. Esse diretório só pode ser criado na aba “arquivos/files” do NetBeans. Esse diretório deve se chamar “antlr4” e deve ter a mesma estrutura de diretórios que os pacotes Java. Observe que é necessário alternar para a aba “arquivos/files”:

- Ao retornar para a aba “projetos/projects” a nova estrutura de diretórios irá aparecer dentro de “other sources”

- O conteúdo do arquivo é o seguinte (é a mesma gramática que o analisador semântico)

```
grammar Alguma;

TIPO_VAR: 'INTEIRO' | 'REAL';
NUMINT: ('0'..'9')+;
NUMREAL: ('0'..'9')+ '.' ('0'..'9')+;
CADEIA:'\'' ( ESC_SEQ | ~('\''|'\\') )* '\'';
fragment
ESC_SEQ: '\\\'';
OP_ARIT1: '+' | '-';
OP_ARIT2: '*' | '/';
OP_REL: '>' | '>=' | '<' | '<=' | '<>' | '=';
OP_BOOL: 'E' | 'OU';
VARIAVEL: ('a'..'z'|'A'..'Z') ('a'..'z'|'A'..'Z'|'0'..'9')*;
COMENTARIO:'%' ~('\n'|'\r')* '\r'? '\n' -> skip;
WS: ( ' ' |'\t' | '\r' | '\n') -> skip;
	
programa: ':' 'DECLARACOES' declaracao* ':' 'ALGORITMO' comando* EOF;
declaracao: VARIAVEL ':' TIPO_VAR;
expressaoAritmetica: termoAritmetico (OP_ARIT1 termoAritmetico)*;
termoAritmetico: fatorAritmetico (OP_ARIT2 fatorAritmetico)*;
fatorAritmetico: NUMINT | NUMREAL | VARIAVEL | '(' expressaoAritmetica ')';
expressaoRelacional: termoRelacional (OP_BOOL termoRelacional)*;
termoRelacional: expressaoAritmetica OP_REL expressaoAritmetica | '(' expressaoRelacional ')';
comando: comandoAtribuicao | comandoEntrada | comandoSaida | comandoCondicao | comandoRepeticao |	subAlgoritmo;
comandoAtribuicao: 'ATRIBUIR' expressaoAritmetica 'A' VARIAVEL;
comandoEntrada: 'LER' VARIAVEL;
comandoSaida: 'IMPRIMIR' (expressaoAritmetica | CADEIA);
comandoCondicao: 'SE' expressaoRelacional 'ENTAO' comando ('SENAO' comando)?;
comandoRepeticao: 'ENQUANTO' expressaoRelacional comando;
subAlgoritmo: 'INICIO' comando* 'FIM';
```

4. Compilar o projeto e criar as seguintes classes (são as mesmas do analisador semântico que vamos reaproveitar)

- ```TabelaDeSimbolos```

```java
package br.compiladores.compiladores.alguma.gerador;

import java.util.HashMap;
import java.util.Map;

public class TabelaDeSimbolos {
    public enum TipoAlguma {
        INTEIRO,
        REAL,
        INVALIDO
    }
    
    class EntradaTabelaDeSimbolos {
        String nome;
        TipoAlguma tipo;

        private EntradaTabelaDeSimbolos(String nome, TipoAlguma tipo) {
            this.nome = nome;
            this.tipo = tipo;
        }
  `  }
    
    private final Map<String, EntradaTabelaDeSimbolos> tabela;
    
    public TabelaDeSimbolos() {
        this.tabela = new HashMap<>();
    }
    
    public void adicionar(String nome, TipoAlguma tipo) {
        tabela.put(nome, new EntradaTabelaDeSimbolos(nome, tipo));
    }
    
    public boolean existe(String nome) {
        return tabela.containsKey(nome);
    }
    
    public TipoAlguma verificar(String nome) {
        return tabela.get(nome).tipo;
    }
}
```

- ```AlgumaSemanticoUtils```

```java
package br.compiladores.compiladores.alguma.gerador;

import java.util.ArrayList;
import java.util.List;
import org.antlr.v4.runtime.Token;

public class AlgumaSemanticoUtils {
    public static List<String> errosSemanticos = new ArrayList<>();
    
    public static void adicionarErroSemantico(Token t, String mensagem) {
        int linha = t.getLine();
        int coluna = t.getCharPositionInLine();
        errosSemanticos.add(String.format("Erro %d:%d - %s", linha, coluna, mensagem));
    }
    
    public static TabelaDeSimbolos.TipoAlguma verificarTipo(TabelaDeSimbolos tabela, AlgumaParser.ExpressaoAritmeticaContext ctx) {
        TabelaDeSimbolos.TipoAlguma ret = null;
        for (var ta : ctx.termoAritmetico()) {
            TabelaDeSimbolos.TipoAlguma aux = verificarTipo(tabela, ta);
            if (ret == null) {
                ret = aux;
            } else if (ret != aux && aux != TabelaDeSimbolos.TipoAlguma.INVALIDO) {
                adicionarErroSemantico(ctx.start, "Expressão " + ctx.getText() + " contém tipos incompatíveis");
                ret = TabelaDeSimbolos.TipoAlguma.INVALIDO;
            }
        }

        return ret;
    }

    public static TabelaDeSimbolos.TipoAlguma verificarTipo(TabelaDeSimbolos tabela, AlgumaParser.TermoAritmeticoContext ctx) {
        TabelaDeSimbolos.TipoAlguma ret = null;

        for (var fa : ctx.fatorAritmetico()) {
            TabelaDeSimbolos.TipoAlguma aux = verificarTipo(tabela, fa);
            if (ret == null) {
                ret = aux;
            } else if (ret != aux && aux != TabelaDeSimbolos.TipoAlguma.INVALIDO) {
                adicionarErroSemantico(ctx.start, "Termo " + ctx.getText() + " contém tipos incompatíveis");
                ret = TabelaDeSimbolos.TipoAlguma.INVALIDO;
            }
        }
        return ret;
    }

    public static TabelaDeSimbolos.TipoAlguma verificarTipo(TabelaDeSimbolos tabela, AlgumaParser.FatorAritmeticoContext ctx) {
        if (ctx.NUMINT() != null) {
            return TabelaDeSimbolos.TipoAlguma.INTEIRO;
        }
        if (ctx.NUMREAL() != null) {
            return TabelaDeSimbolos.TipoAlguma.REAL;
        }
        if (ctx.VARIAVEL() != null) {
            String nomeVar = ctx.VARIAVEL().getText();
            if (!tabela.existe(nomeVar)) {
                adicionarErroSemantico(ctx.VARIAVEL().getSymbol(), "Variável " + nomeVar + " não foi declarada antes do uso");
                return TabelaDeSimbolos.TipoAlguma.INVALIDO;
            }
            return verificarTipo(tabela, nomeVar);
        }
        // se não for nenhum dos tipos acima, só pode ser uma expressão
        // entre parêntesis
        return verificarTipo(tabela, ctx.expressaoAritmetica());
    }
    
    public static TabelaDeSimbolos.TipoAlguma verificarTipo(TabelaDeSimbolos tabela, String nomeVar) {
        return tabela.verificar(nomeVar);
    }
}
```

- ```AlgumaSemantico```

```java
package br.compiladores.compiladores.alguma.gerador;

import br.compiladores.compiladores.alguma.gerador.TabelaDeSimbolos.TipoAlguma;

public class AlgumaSemantico extends AlgumaBaseVisitor<Void> {

    TabelaDeSimbolos tabela;

    @Override
    public Void visitPrograma(AlgumaParser.ProgramaContext ctx) {
        tabela = new TabelaDeSimbolos();
        return super.visitPrograma(ctx);
    }

    @Override
    public Void visitDeclaracao(AlgumaParser.DeclaracaoContext ctx) {
        String nomeVar = ctx.VARIAVEL().getText();
        String strTipoVar = ctx.TIPO_VAR().getText();
        TipoAlguma tipoVar = TipoAlguma.INVALIDO;
        switch (strTipoVar) {
            case "INTEIRO":
                tipoVar = TipoAlguma.INTEIRO;
                break;
            case "REAL":
                tipoVar = TipoAlguma.REAL;
                break;
            default:
                // Nunca irá acontecer, pois o analisador sintático
                // não permite
                break;
        }

        // Verificar se a variável já foi declarada
        if (tabela.existe(nomeVar)) {
            AlgumaSemanticoUtils.adicionarErroSemantico(ctx.VARIAVEL().getSymbol(), "Variável " + nomeVar + " já existe");
        } else {
            tabela.adicionar(nomeVar, tipoVar);
        }

        return super.visitDeclaracao(ctx);
    }

    @Override
    public Void visitComandoAtribuicao(AlgumaParser.ComandoAtribuicaoContext ctx) {
        TipoAlguma tipoExpressao = AlgumaSemanticoUtils.verificarTipo(tabela, ctx.expressaoAritmetica());
        if (tipoExpressao != TipoAlguma.INVALIDO) {
            String nomeVar = ctx.VARIAVEL().getText();
            if (!tabela.existe(nomeVar)) {
                AlgumaSemanticoUtils.adicionarErroSemantico(ctx.VARIAVEL().getSymbol(), "Variável " + nomeVar + " não foi declarada antes do uso");
            } else {
                TipoAlguma tipoVariavel = AlgumaSemanticoUtils.verificarTipo(tabela, nomeVar);
                if (tipoVariavel != tipoExpressao && tipoExpressao != TipoAlguma.INVALIDO) {
                    AlgumaSemanticoUtils.adicionarErroSemantico(ctx.VARIAVEL().getSymbol(), "Tipo da variável " + nomeVar + " não é compatível com o tipo da expressão");
                }
            }
        }
        return super.visitComandoAtribuicao(ctx);
    }

    @Override
    public Void visitComandoEntrada(AlgumaParser.ComandoEntradaContext ctx) {
        String nomeVar = ctx.VARIAVEL().getText();
        if (!tabela.existe(nomeVar)) {
            AlgumaSemanticoUtils.adicionarErroSemantico(ctx.VARIAVEL().getSymbol(), "Variável " + nomeVar + " não foi declarada antes do uso");
        }
        return super.visitComandoEntrada(ctx);
    }

    @Override
    public Void visitExpressaoAritmetica(AlgumaParser.ExpressaoAritmeticaContext ctx) {
        AlgumaSemanticoUtils.verificarTipo(tabela, ctx);
        return super.visitExpressaoAritmetica(ctx);
    }
}
```

5. Vamos agora criar o gerador (um visitante)

```java
package br.compiladores.compiladores.alguma.gerador;

import br.compiladores.compiladores.alguma.gerador.TabelaDeSimbolos.TipoAlguma;

public class AlgumaGeradorC extends AlgumaBaseVisitor<Void> {

    StringBuilder saida;
    TabelaDeSimbolos tabela;

    public AlgumaGeradorC() {
        saida = new StringBuilder();
        this.tabela = new TabelaDeSimbolos();
    }

    @Override
    public Void visitPrograma(AlgumaParser.ProgramaContext ctx) {
        saida.append("#include <stdio.h>\n");
        saida.append("#include <stdlib.h>\n");
        saida.append("\n");
        ctx.declaracao().forEach(dec -> visitDeclaracao(dec));
        saida.append("\n");
        saida.append("int main() {\n");
        ctx.comando().forEach(cmd -> visitComando(cmd));
        saida.append("}\n");
        return null;
    }

    @Override
    public Void visitDeclaracao(AlgumaParser.DeclaracaoContext ctx) {
        String nomeVar = ctx.VARIAVEL().getText();
        String strTipoVar = ctx.TIPO_VAR().getText();
        TabelaDeSimbolos.TipoAlguma tipoVar = TabelaDeSimbolos.TipoAlguma.INVALIDO;
        switch (strTipoVar) {
            case "INTEIRO":
                tipoVar = TabelaDeSimbolos.TipoAlguma.INTEIRO;
                strTipoVar = "int";
                break;
            case "REAL":
                tipoVar = TabelaDeSimbolos.TipoAlguma.REAL;
                strTipoVar = "float";
                break;
            default:
                // Nunca irá acontecer, pois o analisador sintático
                // não permite
                break;
        }
        // Podemos adicionar na tabela de símbolos sem verificar
        // pois a análise semântica já fez as verificações
        tabela.adicionar(nomeVar, tipoVar);
        saida.append(strTipoVar + " " + nomeVar + ";\n");
        return null;
    }

    @Override
    public Void visitComandoAtribuicao(AlgumaParser.ComandoAtribuicaoContext ctx) {
        saida.append(ctx.VARIAVEL().getText() + " = ");
        visitExpressaoAritmetica(ctx.expressaoAritmetica());
        saida.append(";\n");
        return null;
    }

    @Override
    public Void visitComandoCondicao(AlgumaParser.ComandoCondicaoContext ctx) {
        saida.append("if(");
        visitExpressaoRelacional(ctx.expressaoRelacional());
        saida.append(")\n");
        visitComando(ctx.comando(0));
        if (ctx.comando().size() > 1) { // tem else
            saida.append("else\n");
            visitComando(ctx.comando(1));
        }
        return null;
    }

    @Override
    public Void visitComandoEntrada(AlgumaParser.ComandoEntradaContext ctx) {
        String nomeVar = ctx.VARIAVEL().getText();
        TipoAlguma tipoVariavel = AlgumaSemanticoUtils.verificarTipo(tabela, nomeVar);
        String aux = "";
        switch (tipoVariavel) {
            case INTEIRO:
                aux = "%d";
                break;
            case REAL:
                aux = "%f";
                break;
        }
        saida.append("scanf(\"" + aux + "\", &" + nomeVar + ");\n");
        return null;
    }

    @Override
    public Void visitComandoRepeticao(AlgumaParser.ComandoRepeticaoContext ctx) {
        saida.append("while(");
        visitExpressaoRelacional(ctx.expressaoRelacional());
        saida.append(")\n");
        visitComando(ctx.comando());
        return null;
    }

    @Override
    public Void visitComandoSaida(AlgumaParser.ComandoSaidaContext ctx) {
        if (ctx.CADEIA() != null) {
            String aux = ctx.CADEIA().getText();
            aux = aux.substring(1, aux.length() - 1);
            saida.append("printf(\"" + aux + "\\n\");\n");
        } else {
            TipoAlguma tipoExpressao = AlgumaSemanticoUtils.verificarTipo(tabela, ctx.expressaoAritmetica());
            String aux = "";
            switch (tipoExpressao) {
                case INTEIRO:
                    aux = "%d";
                    break;
                case REAL:
                    aux = "%f";
                    break;
            }
            saida.append("printf(\"" + aux + "\\n\",");
            visitExpressaoAritmetica(ctx.expressaoAritmetica());
            saida.append(");\n");
        }
        return null;
    }

    @Override
    public Void visitSubAlgoritmo(AlgumaParser.SubAlgoritmoContext ctx) {
        saida.append("{\n");
        ctx.comando().forEach(cmd -> visitComando(cmd));
        saida.append("}\n");
        return null;
    }

    @Override
    public Void visitExpressaoAritmetica(AlgumaParser.ExpressaoAritmeticaContext ctx) {
        visitTermoAritmetico(ctx.termoAritmetico(0));
        for (int i = 0; i < ctx.OP_ARIT1().size(); i++) {
            saida.append(" " + ctx.OP_ARIT1(i).getText() + " ");
            visitTermoAritmetico(ctx.termoAritmetico(i + 1));
        }
        return null;
    }

    @Override
    public Void visitTermoAritmetico(AlgumaParser.TermoAritmeticoContext ctx) {
        visitFatorAritmetico(ctx.fatorAritmetico(0));
        for (int i = 0; i < ctx.OP_ARIT2().size(); i++) {
            saida.append(" " + ctx.OP_ARIT2(i).getText() + " ");
            visitFatorAritmetico(ctx.fatorAritmetico(i + 1));
        }
        return null;
    }

    @Override
    public Void visitFatorAritmetico(AlgumaParser.FatorAritmeticoContext ctx) {
        if (ctx.NUMINT() != null) {
            saida.append(ctx.NUMINT().getText());
        } else if (ctx.NUMREAL() != null) {
            saida.append(ctx.NUMREAL().getText());
        } else if (ctx.VARIAVEL() != null) {
            saida.append(ctx.VARIAVEL().getText());
        } else {
            saida.append("(");
            visitExpressaoAritmetica(ctx.expressaoAritmetica());
            saida.append(")");
        }
        return null;
    }

    @Override
    public Void visitExpressaoRelacional(AlgumaParser.ExpressaoRelacionalContext ctx) {
        visitTermoRelacional(ctx.termoRelacional(0));
        for (int i = 0; i < ctx.OP_BOOL().size(); i++) {
            String aux = null;
            if (ctx.OP_BOOL(i).getText().equals("E")) {
                aux = "&&";
            } else {
                aux = "||";
            }
            saida.append(" " + aux + " ");
            visitTermoRelacional(ctx.termoRelacional(i + 1));
        }
        return null;
    }

    @Override
    public Void visitTermoRelacional(AlgumaParser.TermoRelacionalContext ctx) {
        if (ctx.expressaoRelacional() != null) {
            saida.append("(");
            visitExpressaoRelacional(ctx.expressaoRelacional());
            saida.append(")");
        } else {
            visitExpressaoAritmetica(ctx.expressaoAritmetica(0));
            String aux = ctx.OP_REL().getText();
            if (aux.equals("<>")) {
                aux = "!=";
            } else if (aux.equals("=")) {
                aux = "==";
            }
            saida.append(" " + aux + " ");
            visitExpressaoAritmetica(ctx.expressaoAritmetica(1));
        }
        return null;
    }
}
```

6. Criar a classe Principal:

```java
package br.compiladores.compiladores.alguma.gerador;

import br.compiladores.compiladores.alguma.gerador.AlgumaParser.ProgramaContext;
import java.io.IOException;
import java.io.PrintWriter;
import org.antlr.v4.runtime.CharStream;
import org.antlr.v4.runtime.CharStreams;
import org.antlr.v4.runtime.CommonTokenStream;

public class Principal {
    public static void main(String args[]) throws IOException {
        CharStream cs = CharStreams.fromFileName(args[0]);
        AlgumaLexer lexer = new AlgumaLexer(cs);
        CommonTokenStream tokens = new CommonTokenStream(lexer);
        AlgumaParser parser = new AlgumaParser(tokens);
        ProgramaContext arvore = parser.programa();
        AlgumaSemantico as = new AlgumaSemantico();
        as.visitPrograma(arvore);
        AlgumaSemanticoUtils.errosSemanticos.forEach((s) -> System.out.println(s));
        
        if(AlgumaSemanticoUtils.errosSemanticos.isEmpty()) {
            AlgumaGeradorC agc = new AlgumaGeradorC();
            agc.visitPrograma(arvore);
            try(PrintWriter pw = new PrintWriter(args[1])) {
                pw.print(agc.saida.toString());
            }
        }
    }
}
```

7. Compilar e rodar, abrindo em um terminal (não esquecer de substituir os caminhos)

```sh
java -jar ~/NetBeansProjects/alguma-gerador/target/alguma-gerador-1.0-SNAPSHOT-jar-with-dependencies.jar ~/Desktop/exemplo.txt ~/Desktop/saida.c
```

8. Para testar o código gerado, basta compilar usando gcc e depois rodar

```java
gcc ~/Desktop/saida.c -> irá gerar o arquivo a.out
./a.out
```

