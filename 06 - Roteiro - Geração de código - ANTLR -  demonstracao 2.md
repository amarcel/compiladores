# Construção de Compiladores - Daniel Lucrédio, Helena Caseli, Mário César San Felice e Murilo Naldi
## Geração de Código - Exemplos

### Demonstração 2 - Ambiente de execução P-código e Gerando P-código a partir da linguagem Alguma
---

1. Rodar a máquina P-código:

- Executar em um terminal, dentro da pasta descompactada:

```sh
java -jar pcodemachine.jar
```

2. Testar com diferentes exemplos

```
// 2 * a + (b - 3)
lda 0
rdi
lda 1
rdi
ldc 2
lod 0
mpi
lod 1
ldc 3
sbi
adi
wri
stp
```

```
// Fatorial
lda 0
rdi
lod 0
ldc 0
grt
fjp L1
lda 1
ldc 1
sto
lab L2
lda 1
lod 1
lod 0
mpi
sto
lda 0
lod 0
ldc 1
sbi
sto
lod 0
ldc 0
equ
fjp L2
lod 1
wri
lab L1
stp
```

3. Abrir o projeto da Demonstração 1
4. Vamos precisar modificar um pouco a tabela de símbolos para recordar os endereços na pilha

```diff
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
+        int endereco;

        private EntradaTabelaDeSimbolos(String nome, TipoAlguma tipo) {
            this.nome = nome;
            this.tipo = tipo;
        }

+        private EntradaTabelaDeSimbolos(String nome, int endereco) {
+            this.nome = nome;
+            this.endereco = endereco;
+        }
    }
    
    private final Map<String, EntradaTabelaDeSimbolos> tabela;
    
    public TabelaDeSimbolos() {
        this.tabela = new HashMap<>();
    }
    
    public void adicionar(String nome, TipoAlguma tipo) {
        tabela.put(nome, new EntradaTabelaDeSimbolos(nome, tipo));
    }

+    public void adicionar(String nome, int endereco) {
+        tabela.put(nome, new EntradaTabelaDeSimbolos(nome, endereco));
+    }
    
    public boolean existe(String nome) {
        return tabela.containsKey(nome);
    }
    
    public TipoAlguma verificar(String nome) {
        return tabela.get(nome).tipo;
    }
    
+    public int verificarEndereco(String nome) {
+        return tabela.get(nome).endereco;
+    }
}
```

5. Adicionar um novo visitante para gerar P-código (desta vez vamos usar o retorno dos métodos para gerar o código):

```java
package br.compiladores.compiladores.alguma.gerador;

public class AlgumaGeradorPcodigo extends AlgumaBaseVisitor<String> {

    TabelaDeSimbolos tabela = new TabelaDeSimbolos();
    int enderecoAtual = 0;
    int label = 0;

    @Override
    public String visitPrograma(AlgumaParser.ProgramaContext ctx) {
        String pcod = "";
        ctx.declaracao().forEach(dec -> visitDeclaracao(dec));
        for (var c : ctx.comando()) {
            pcod += visitComando(c);
        }
        pcod += "stp\n";
        return pcod;
    }

    @Override
    public String visitDeclaracao(AlgumaParser.DeclaracaoContext ctx) {
        tabela.adicionar(ctx.VARIAVEL().getText(), enderecoAtual++);
        return null;
    }

    @Override
    public String visitExpressaoAritmetica(AlgumaParser.ExpressaoAritmeticaContext ctx) {
        String pcod = "";
        pcod += visitTermoAritmetico(ctx.termoAritmetico(0));
        for (int i = 1; i < ctx.termoAritmetico().size(); i++) {
            pcod += visitTermoAritmetico(ctx.termoAritmetico(i));
            if (ctx.OP_ARIT1(i - 1).getText().equals("+")) {
                pcod += "adi\n";
            } else if (ctx.OP_ARIT1(i - 1).getText().equals("-")) {
                pcod += "sbi\n";
            }
        }
        return pcod;
    }

    @Override
    public String visitTermoAritmetico(AlgumaParser.TermoAritmeticoContext ctx) {
        String pcod = "";
        pcod += visitFatorAritmetico(ctx.fatorAritmetico(0));
        for (int i = 1; i < ctx.fatorAritmetico().size(); i++) {
            pcod += visitFatorAritmetico(ctx.fatorAritmetico(i));
            if (ctx.OP_ARIT2(i - 1).getText().equals("*")) {
                pcod += "mpi\n";
            } else if (ctx.OP_ARIT2(i - 1).getText().equals("/")) {
                pcod += "dvi\n";
            }
        }
        return pcod;
    }

    @Override
    public String visitFatorAritmetico(AlgumaParser.FatorAritmeticoContext ctx) {
        if (ctx.NUMINT() != null) {
            return "ldc " + ctx.NUMINT().getText() + "\n";
        } else if (ctx.NUMREAL() != null) {
            return "ldc " + ctx.NUMREAL().getText() + "\n";
        } else if (ctx.VARIAVEL() != null) {
            int endereco = tabela.verificarEndereco(ctx.VARIAVEL().getText());
            return "lod " + endereco + "\n";
        } else {
            return visitExpressaoAritmetica(ctx.expressaoAritmetica());
        }
    }

    @Override
    public String visitExpressaoRelacional(AlgumaParser.ExpressaoRelacionalContext ctx) {
        String pcod = visitTermoRelacional(ctx.termoRelacional(0));
        for (int i = 1; i < ctx.termoRelacional().size(); i++) {
            pcod += visitTermoRelacional(ctx.termoRelacional(i));
            if (ctx.OP_BOOL(i - 1).getText().equals("E")) {
                pcod += "and\n";
            } else if (ctx.OP_BOOL(i - 1).getText().equals("OU")) {
                pcod += "or\n";
            }

        }
        return pcod;
    }

    @Override
    public String visitTermoRelacional(AlgumaParser.TermoRelacionalContext ctx) {
        String pcod = "";
        if (ctx.expressaoRelacional() != null) {
            pcod = visitExpressaoRelacional(ctx.expressaoRelacional());

        } else {
            pcod += visitExpressaoAritmetica(ctx.expressaoAritmetica(0)) + visitExpressaoAritmetica(ctx.expressaoAritmetica(1));
            switch (ctx.OP_REL().getText()) {
                case ">":
                    pcod += "grt\n";
                    break;
                case ">=":
                    pcod += "gte\n";
                    break;
                case "<":
                    pcod += "let\n";
                    break;
                case "<=":
                    pcod += "lte\n";
                    break;
                case "<>":
                    pcod += "neq\n";
                    break;
                case "=":
                    pcod += "equ\n";
                    break;
                default:
                    break;
            }
        }

        return pcod;
    }

    @Override
    public String visitComando(AlgumaParser.ComandoContext ctx) {
        if (ctx.comandoAtribuicao() != null) {
            return visitComandoAtribuicao(ctx.comandoAtribuicao());
        } else if (ctx.comandoEntrada() != null) {
            return visitComandoEntrada(ctx.comandoEntrada());
        } else if (ctx.comandoSaida() != null) {
            return visitComandoSaida(ctx.comandoSaida());
        } else if (ctx.comandoCondicao() != null) {
            return visitComandoCondicao(ctx.comandoCondicao());
        } else if (ctx.comandoRepeticao() != null) {
            return visitComandoRepeticao(ctx.comandoRepeticao());
        } else if (ctx.subAlgoritmo() != null) {
            return visitSubAlgoritmo(ctx.subAlgoritmo());
        }
        return null;
    }

    @Override
    public String visitComandoAtribuicao(AlgumaParser.ComandoAtribuicaoContext ctx) {
        int endereco = tabela.verificarEndereco(ctx.VARIAVEL().getText());
        return "lda " + endereco + "\n"
                + visitExpressaoAritmetica(ctx.expressaoAritmetica())
                + "sto\n";
    }

    @Override
    public String visitComandoEntrada(AlgumaParser.ComandoEntradaContext ctx) {
        int endereco = tabela.verificarEndereco(ctx.VARIAVEL().getText());
        return "lda " + endereco + "\n"
                + "rdi\n";
    }

    @Override
    public String visitComandoSaida(AlgumaParser.ComandoSaidaContext ctx) {
        if (ctx.expressaoAritmetica() != null) {
            return visitExpressaoAritmetica(ctx.expressaoAritmetica()) +
                    "wri\n";
        }
        return "";
    }

    @Override
    public String visitComandoCondicao(AlgumaParser.ComandoCondicaoContext ctx) {
        String pcod;

        int label1 = label++;
        pcod = visitExpressaoRelacional(ctx.expressaoRelacional());
        pcod += "fjp L" + label1 + "\n";
        pcod += visitComando(ctx.comando(0));
        if (ctx.comando().size() > 1) {
            int label2 = label++;
            pcod += "ujp L" + label2 + "\n";
            pcod += "lab L" + label1 + "\n";
            pcod += visitComando(ctx.comando(1));
            pcod += "lab L" + label2 + "\n";
        } else {
            pcod += "lab L" + label1 + "\n";
        }

        return pcod;
    }

    @Override
    public String visitComandoRepeticao(AlgumaParser.ComandoRepeticaoContext ctx) {
        String pcod;
        int label1 = label++;
        int label2 = label++;
        pcod = "lab L" + label1 + "\n";
        pcod += visitExpressaoRelacional(ctx.expressaoRelacional());
        pcod += "fjp L" + label2 + "\n";
        pcod += visitComando(ctx.comando());
        pcod += "ujp L" + label1 + "\n";
        pcod += "lab L" + label2 + "\n";

        return pcod;
    }

    @Override
    public String visitSubAlgoritmo(AlgumaParser.SubAlgoritmoContext ctx) {
        String pcod = "";
        for (var c : ctx.comando()) {
            pcod += visitComando(c);
        }
        return pcod;
    }
}
```

6. Modificar a classe Principal para gerar P-código, além de C:

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
            
            AlgumaGeradorPcodigo agp = new AlgumaGeradorPcodigo();
            String pcod = agp.visitPrograma(arvore);
            try(PrintWriter pw = new PrintWriter(args[2])) {
                pw.print(pcod);
            }
        }
    }
}
```

7. Testar, gerando o código e colocando na pcodemachine para rodar
8. Outros algoritmos para testar

```
:DECLARACOES
argumento:INTEIRO
fatorial:INTEIRO

:ALGORITMO
% Calcula o fatorial de um número inteiro
LER argumento
ATRIBUIR argumento A fatorial
SE argumento = 0 ENTAO ATRIBUIR 1 A fatorial
ENQUANTO argumento > 1
   INICIO
      ATRIBUIR fatorial * (argumento - 1) A fatorial
      ATRIBUIR argumento - 1 A argumento
   FIM
IMPRIMIR fatorial
```



```
:DECLARACOES
numero1:INTEIRO
numero2:INTEIRO
numero3:INTEIRO
aux:INTEIRO

:ALGORITMO
% Coloca 3 números em ordem crescente
LER numero1
LER numero2
LER numero3
SE numero1 > numero2 ENTAO
   INICIO
      ATRIBUIR numero2 A aux
      ATRIBUIR numero1 A numero2
      ATRIBUIR aux A numero1
   FIM 
FIMSE
SE numero1 > numero3 ENTAO
   INICIO
      ATRIBUIR numero3 A aux
      ATRIBUIR numero1 A numero3
      ATRIBUIR aux A numero1
   FIM
FIMSE
SE numero2 > numero3 ENTAO
   INICIO
      ATRIBUIR numero3 A aux
      ATRIBUIR numero2 A numero3
      ATRIBUIR aux A numero2
   FIM
FIMSE
IMPRIMIR numero1
IMPRIMIR numero2
IMPRIMIR numero3
```

