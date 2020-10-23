# Construção de Compiladores
## Análisador Sintático
## Fonte: Daniel Lucrédio, Helena Caseli, Mário César San Felice e Murilo Naldi

### Demonstração 1 – Analisador sintático preditivo de descendência recursiva “na mão”
---

1. Criar um novo arquivo, no Desktop, com um programa de exemplo

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
      ATRIBUIR 2+3-4+5-6*5-1 A aux
      ATRIBUIR numero1 A numero2
      ATRIBUIR aux A numero1
   FIM 
SE numero1 > numero3 E numero2 <= numero4 E numero1 > 3 OU numero2 <> numero4 ENTAO
   INICIO
      ATRIBUIR (numero3) A aux
      ATRIBUIR numero1 A numero3
      ATRIBUIR aux A numero1
   FIM
SE numero2 > numero3 ENTAO
   INICIO
      ATRIBUIR numero3 A aux
      ATRIBUIR numero2 A numero3
      ATRIBUIR aux A numero2
   FIM
IMPRIMIR numero1
IMPRIMIR numero2
IMPRIMIR numero3
```

2. Abrir o NetBeans, e abrir projeto Java “AlgumaLex”
3. Criar novo projeto Java (com Ant) “AlgumaParser”
4. Adicionar dependência do “AlgumaParser” para o “AlgumaLex”
5. Criar um arquivo com a gramática

```
programa : ':' 'DECLARACOES' listaDeclaracoes ':' 'ALGORITMO' listaComandos;
listaDeclaracoes : declaracao listaDeclaracoes | declaracao;
declaracao : VARIAVEL ':' tipoVar;
tipoVar : 'INTEIRO' | 'REAL';
expressaoAritmetica : expressaoAritmetica '+' termoAritmetico | expressaoAritmetica '-' termoAritmetico | termoAritmetico;
termoAritmetico : termoAritmetico '*' fatorAritmetico | termoAritmetico '/' fatorAritmetico | fatorAritmetico;
fatorAritmetico : NUMINT | NUMREAL | VARIAVEL | '(' expressaoAritmetica ')'
expressaoRelacional : expressaoRelacional operadorBooleano termoRelacional | termoRelacional;
termoRelacional : expressaoAritmetica OP_REL expressaoAritmetica | '(' expressaoRelacional ')';
operadorBooleano : 'E' | 'OU';
listaComandos : comando listaComandos | comando;
comando : comandoAtribuicao | comandoEntrada | comandoSaida | comandoCondicao | comandoRepeticao | subAlgoritmo;
comandoAtribuicao : 'ATRIBUIR' expressaoAritmetica 'A' VARIAVEL;
comandoEntrada : 'LER' VARIAVEL;
comandoSaida : 'IMPRIMIR'  (VARIAVEL | CADEIA);
comandoCondicao : 'SE' expressaoRelacional 'ENTAO' comando | 'SE' expressaoRelacional 'ENTAO' comando 'SENAO' comando;
comandoRepeticao : 'ENQUANTO' expressaoRelacional comando;
subAlgoritmo : 'INICIO' listaComandos 'FIM';
```

6. Criar uma classe algumaparser.AlgumaParser

```java
package algumaparser;

import algumalex.AlgumaLexico;
import algumalex.TipoToken;
import algumalex.Token;
import java.util.ArrayList;
import java.util.List;

public class AlgumaParser {

    private final static int TAMANHO_BUFFER = 10;
    List<Token> bufferTokens;
    AlgumaLexico lex;
    boolean chegouNoFim = false;

    public AlgumaParser(AlgumaLexico lex) {
        this.lex = lex;
        bufferTokens = new ArrayList<Token>();
        lerToken();
    }

    private void lerToken() {
        if (bufferTokens.size() > 0) {
            bufferTokens.remove(0);
        }
        while (bufferTokens.size() < TAMANHO_BUFFER && !chegouNoFim) {
            Token proximo = lex.proximoToken();
            bufferTokens.add(proximo);
            if (proximo.nome == TipoToken.Fim) {
                chegouNoFim = true;
            }
        }
        System.out.println("Lido:  " + lookahead(1));
    }

    void match(TipoToken tipo) {
        if (lookahead(1).nome == tipo) {
            System.out.println("Match: " + lookahead(1));
            lerToken();
        } else {
            erroSintatico(tipo.toString());
        }
    }

    Token lookahead(int k) {
        if (bufferTokens.isEmpty()) {
            return null;
        }
        if (k - 1 >= bufferTokens.size()) {
            return bufferTokens.get(bufferTokens.size() - 1);
        }
        return bufferTokens.get(k - 1);
    }

    void erroSintatico(String... tokensEsperados) {
        String mensagem = "Erro sintático: esperando um dos seguintes (";
        for(int i=0;i<tokensEsperados.length;i++) {
            mensagem += tokensEsperados[i];
            if(i<tokensEsperados.length-1)
                mensagem += ",";
        }
        mensagem += "), mas foi encontrado " + lookahead(1);
        throw new RuntimeException(mensagem);
    }
}
```

7. Copiar a gramática para dentro do arquivo AlgumaParser.java, comentando suas linhas
8. Transformar cada linha em um método

```java
    //programa : ':' 'DECLARACOES' listaDeclaracoes ':' 'ALGORITMO' listaComandos;
    public void programa() {
        match(TipoToken.Delim);
        match(TipoToken.PCDeclaracoes);
        listaDeclaracoes();
        match(TipoToken.Delim);
        match(TipoToken.PCAlgoritmo);
        listaComandos();
        match(TipoToken.Fim);
    }

    //listaDeclaracoes : declaracao listaDeclaracoes | declaracao;
    void listaDeclaracoes() {
        // usaremos lookahead(4)
        // Mas daria para fatorar à esquerda
        // Veja na lista de comandos um exemplo
        if (lookahead(4).nome == TipoToken.Delim) {
            declaracao();
        } else if (lookahead(4).nome == TipoToken.Var) {
            declaracao();
            listaDeclaracoes();
        } else {
            erroSintatico(TipoToken.Delim.toString(), TipoToken.Var.toString());
        }
    }

    //declaracao : VARIAVEL ':' tipoVar;
    // A) PENSE E PROGRAME

    //tipoVar : 'INTEIRO' | 'REAL';
    // B) PENSE E PROGRAME

    //expressaoAritmetica : expressaoAritmetica '+' termoAritmetico | expressaoAritmetica '-' termoAritmetico | termoAritmetico;
    // fatorar à esquerda:
    // expressaoAritmetica : expressaoAritmetica ('+' termoAritmetico | '-' termoAritmetico) | termoAritmetico;
    // fatorar não é suficiente, pois causa loop infinito
    // remover a recursão à esquerda
    // expressaoAritmetica : termoAritmetico expressaoAritmetica2
    // expressaoAritmetica2 : ('+' termoAritmetico | '-' termoAritmetico) expressaoAritmetica2 | <<vazio>>
    void expressaoAritmetica() {
        termoAritmetico();
        expressaoAritmetica2();
    }

    void expressaoAritmetica2() {
        if (lookahead(1).nome == TipoToken.OpAritSoma || lookahead(1).nome == TipoToken.OpAritSub) {
            expressaoAritmetica2SubRegra1();
            expressaoAritmetica2();
        } else { // vazio
        }
    }

    void expressaoAritmetica2SubRegra1() {
        if (lookahead(1).nome == TipoToken.OpAritSoma) {
            match(TipoToken.OpAritSoma);
            termoAritmetico();
        } else if (lookahead(1).nome == TipoToken.OpAritSub) {
            match(TipoToken.OpAritSub);
            termoAritmetico();
        } else {
            erroSintatico("+","-");
        }
    }

    //termoAritmetico : termoAritmetico '*' fatorAritmetico | termoAritmetico '/' fatorAritmetico | fatorAritmetico;
    // também precisa fatorar à esquerda e eliminar recursão à esquerda
    // termoAritmetico : fatorAritmetico termoAritmetico2
    // termoAritmetico2 : ('*' fatorAritmetico | '/' fatorAritmetico) termoAritmetico2 | <<vazio>>
    // C) PENSE E PROGRAME

    //fatorAritmetico : NUMINT | NUMREAL | VARIAVEL | '(' expressaoAritmetica ')'
    // D) PENSE E PROGRAME

    //expressaoRelacional : expressaoRelacional operadorBooleano termoRelacional | termoRelacional;
    // Precisa eliminar a recursão à esquerda
    // expressaoRelacional : termoRelacional expressaoRelacional2;
    // expressaoRelacional2 : operadorBooleano termoRelacional expressaoRelacional2 | <<vazio>>
    void expressaoRelacional() {
        termoRelacional();
        expressaoRelacional2();
    }

    void expressaoRelacional2() {
        if (lookahead(1).nome == TipoToken.OpBoolE || lookahead(1).nome == TipoToken.OpBoolOu) {
            operadorBooleano();
            termoRelacional();
            expressaoRelacional2();
        } else { // vazio
        }
    }

    //termoRelacional : expressaoAritmetica OP_REL expressaoAritmetica | '(' expressaoRelacional ')';
    void termoRelacional() {
        if (lookahead(1).nome == TipoToken.NumInt
                || lookahead(1).nome == TipoToken.NumReal
                || lookahead(1).nome == TipoToken.Var
                || lookahead(1).nome == TipoToken.AbrePar) {
            // Há um não-determinismo aqui.
            // AbrePar pode ocorrer tanto em expressaoAritmetica como em (expressaoRelacional)
            // Tem uma forma de resolver este problema, mas não usaremos aqui
            // Vamos modificar a linguagem, eliminando a possibilidade
            // de agrupar expressões relacionais com parêntesis
            expressaoAritmetica();
            opRel();
            expressaoAritmetica();
        } else {
            erroSintatico(TipoToken.NumInt.toString(),TipoToken.NumReal.toString(),TipoToken.Var.toString(),"(");
        }
    }

    void opRel() {
        if (lookahead(1).nome == TipoToken.OpRelDif) {
            match(TipoToken.OpRelDif);
        } else if (lookahead(1).nome == TipoToken.OpRelIgual) {
            match(TipoToken.OpRelIgual);
        } else if (lookahead(1).nome == TipoToken.OpRelMaior) {
            match(TipoToken.OpRelMaior);
        } else if (lookahead(1).nome == TipoToken.OpRelMaiorIgual) {
            match(TipoToken.OpRelMaiorIgual);
        } else if (lookahead(1).nome == TipoToken.OpRelMenor) {
            match(TipoToken.OpRelMenor);
        } else if (lookahead(1).nome == TipoToken.OpRelMenorIgual) {
            match(TipoToken.OpRelMenorIgual);
        } else {
            erroSintatico("<>","=",">",">=","<","<=");
        }
    }

    //operadorBooleano : 'E' | 'OU';
    // E) PENSE E PROGRAME

    //listaComandos : comando listaComandos | comando;
    // vamos fatorar à esquerda
    // listaComandos : comando (listaComandos | <<vazio>>)
    void listaComandos() {
        comando();
        listaComandosSubRegra1();
    }

    void listaComandosSubRegra1() {
        if (lookahead(1).nome == TipoToken.PCAtribuir ||
        lookahead(1).nome == TipoToken.PCLer ||
        lookahead(1).nome == TipoToken.PCImprimir ||
        lookahead(1).nome == TipoToken.PCSe ||
        lookahead(1).nome == TipoToken.PCEnquanto ||
        lookahead(1).nome == TipoToken.PCInicio) {
            listaComandos();
        } else {
            // vazio
        }
    }

    //comando : comandoAtribuicao | comandoEntrada | comandoSaida | comandoCondicao | comandoRepeticao | subAlgoritmo;
    // F) PENSE E PROGRAME

    //comandoAtribuicao : 'ATRIBUIR' expressaoAritmetica 'A' VARIAVEL;
    // G) PENSE E PROGRAME

    //comandoEntrada : 'LER' VARIAVEL;
    // H) PENSE E PROGRAME

    //comandoSaida : 'IMPRIMIR'  (VARIAVEL | CADEIA);
    // I) PENSE E PROGRAME

    //comandoCondicao : 'SE' expressaoRelacional 'ENTAO' comando | 'SE' expressaoRelacional 'ENTAO' comando 'SENAO' comando;
    // fatorar à esquerda
    // comandoCondicao : 'SE' expressaoRelacional 'ENTAO' comando ('SENAO' comando | <<vazio>>)
    void comandoCondicao() {
        match(TipoToken.PCSe);
        expressaoRelacional();
        match(TipoToken.PCEntao);
        comando();
        comandoCondicaoSubRegra1();
    }

    void comandoCondicaoSubRegra1() {
        if (lookahead(1).nome == TipoToken.PCSenao) {
            match(TipoToken.PCSenao);
            comando();
        } else {
            // vazio
        }
    }

    //comandoRepeticao : 'ENQUANTO' expressaoRelacional comando;
    // J) PENSE E PROGRAME

    //subAlgoritmo : 'INICIO' listaComandos 'FIM';
    void subAlgoritmo() {
        match(TipoToken.PCInicio);
        listaComandos();
        match(TipoToken.PCFim);
    }
```

9. Criar a classe algumaparser.Principal

```java
public class Principal {
    public static void main(String args[]) {
        AlgumaLexico lex = new AlgumaLexico(args[0]);
        AlgumaParser parser = new AlgumaParser(lex);
        parser.programa();
    }
}
```
10. Compilar e testar

