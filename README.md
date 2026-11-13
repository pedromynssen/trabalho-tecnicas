# Como as LLMs mudam a reconstrução de jogos

## 1. Introdução

No dia 5 de novembro de 2026, a empresa de videogame Nintendo lançou um
remake de *Zelda: Ocarina of Time*, um de seus jogos mais aclamados,
exclusivamente para o Nintendo Switch 2. Em menos de um dia, já existia
um projeto no GitHub que transformava o jogo em um programa nativo para
Windows e Linux, sem emulador.

Com a evolução das inteligências artificiais, diversos projetos que
costumavam demandar uma extensa equipe e diversas horas de trabalho
passaram a custar apenas uma pequena fração do custo em tokens. Para se
ter uma ideia da evolução drástica que esse processo de decompilação
sofreu graças às LLMs, foram necessários 21 meses de trabalho de várias
pessoas para realizar o que o Claude fez em um dia com o *Ocarina of
Time* original, de 1998.

Esses projetos não só dão acesso a esses jogos para mais pessoas como
também abrem portas para os fãs dessas franquias há muito tempo
consolidadas modificarem os jogos, podendo assim inventar novas formas
de jogar clássicos. Isso não é importante só para quem joga, mas também
para programadores em geral, já que conseguimos ver um caso de sucesso
da utilização de LLMs na automatização de um processo que costumava
tomar muito mais tempo e pessoas para torná-lo realidade.

## 2. Mas afinal, como esse processo funciona?

Um jogo de console é distribuído como binário; normalmente, é impossível
para o cliente conseguir entender as estruturas criadas pelos
programadores. Para transformar esses jogos de console em jogos
funcionais para computador, existem três alternativas:

-   **Emulação:** o PC imita o comportamento do console e joga a ROM
    como se fosse no console original.
-   **Recompilação estática:** necessita de uma ferramenta que traduza
    código de máquina para C, que é compilado e executado como um
    programa normal. O código é gerado automaticamente e é difícil de
    ler e entender.
-   **Matching decompilation:** é o processo mais complexo entre os
    três, pois exige que a pessoa crie um código em C que, compilado,
    gere exatamente os mesmos bytes do original.

No caso do Hookshot, port de *Ocarina of Time* (2026) para Windows e
Linux, trata-se de uma recompilação estática. Segundo o design do
projeto, o jogo possui aproximadamente 295.178 funções, cada uma delas
se tornou uma função em C chamada `f_<endereço>`, gerando um total de
1,4 GB de código gerado. O código C gerado não tem nomes: as funções são
identificadas por endereço, e a memória, por deslocamento.

## 3. Refatoração

Refatoração é o processo de mudar a estrutura interna do código sem
alterar o que ele faz para quem o utiliza. Isso inclui renomear
variáveis, dividir funções grandes em outras menores e fazer ajustes que
tornam o código mais limpo.

O que torna a matching decompilation interessante é que ela oferece um
teste muito forte. Se o código refatorado ainda compila para os mesmos
bytes do jogo original, o comportamento certamente permanece sendo o
mesmo. O repositório do *Ocarina of Time* original
([zeldaret/oot](https://github.com/zeldaret/oot)) compara o checksum da
ROM gerada com a original e avisa se algo deixou de bater.

## 4. As habilidades e limitações das IAs nesse espectro

### 4.1 Como ela conseguiu fazer tão rápido?

A recompilação é mais um trabalho braçal: todas as instruções em binário
se transformam em código C difícil de entender. A parte mais difícil é
trabalhar ao redor disso: imitar o sistema do console e traduzir os seus
gráficos.

O Hookshot é um fork do port de Mario Kart World do mesmo autor, e a IA
entrou adaptando essas partes. Para validar a tradução, o projeto tem um
teste robusto: todas as instruções são comparadas com um emulador de CPU
em 44.172 casos, e todas as comparações passam.

Uma semana após o lançamento oficial do jogo e do Hookshot, a lista de
problemas ainda estava cheia de bugs, como texturas sumindo, travamentos
em certas placas AMD e tela preta em GPUs da Intel.

O processo também é novo. O repositório tem um `AGENTS.md` com
instruções para agentes de IA e uma lista de trabalho pontuada por
gravidade e alcance, de onde agentes pegam tarefas e uma pessoa revisa
cada pull request. Em apenas uma semana, houve mais de 2.300 issues e
pull requests.

O mesmo padrão aparece na matching decompilation de *Snowboard Kids 2*
(outro jogo do Nintendo 64) por Chris Lewis, que colocou o Claude em um
ciclo no qual o script compila a cada tentativa e compara os bytes. Em
três semanas usando LLMs, ele conseguiu avançar mais do que em três
meses de trabalho manual. Sem essa conferência constante, a saída da LLM
costuma estar errada.

Em um artigo do LLM4Decompile, o GPT-4o acertou 16% das funções do
teste, e o melhor modelo treinado para isso conseguiu 45%.

### 4.2 Como tornar código ilegível em código aproveitável e escalonável

Chegar em um código funcional não é o fim da jornada. Abaixo está uma
demonstração de como seria o código gerado diretamente por um
decompilador:

``` c
void func_80023A10(void *arg0, int32_t arg1) {
    int32_t temp_v0;
    temp_v0 = *(int16_t *)((char *)arg0 + 0x1C) - arg1;
    if (temp_v0 < 0) { temp_v0 = 0; }
    *(int16_t *)((char *)arg0 + 0x1C) = (int16_t)temp_v0;
    if (temp_v0 == 0) { *(uint8_t *)((char *)arg0 + 0x2A) |= 4; }
}
```

O código teoricamente é funcional, mas é difícil de entender sua função.
Após refatorar ele com a assistência de uma LLM, utilizando um processo
similar ao de outros processos de recompilação, o código ficaria assim:

``` c
#define ACTOR_FLAG_DEAD (1 << 2)

void Actor_ApplyDamage(Actor *actor, int32_t damage) {
    int32_t newHealth = actor->health - damage;
    if (newHealth < 0) { newHealth = 0; }
    actor->health = (int16_t)newHealth;
    if (newHealth == 0) { actor->flags |= ACTOR_FLAG_DEAD; }
}
```

Agora fica mais claro que a função serve para aplicar dano e marcar o
personagem como morto quando sua vida chega a zero. O teste confirma que
o código não quebrou e continua funcionalmente idêntico ao original.
Porém, se a LLM tivesse escolhido outro nome para a função, o teste
ainda passaria, pois os bytes seriam os mesmos.

### 4.3 O que fica faltando nesses novos projetos?

Os ports de jogos do Nintendo Switch 2 citados anteriormente funcionam
da mesma forma que o primeiro exemplo que apresentei. Não existe C
legível, apenas uma tradução automática curada por uma LLM. Para jogar,
isso é o suficiente. Mas, caso você queira fazer melhorias e
modificações no jogo original, o código precisaria ser legível.

À esquerda está um patch real do Hookshot: uma busca e troca de texto em
uma linha do código C gerado, identificada por registrador, endereço e
deslocamento. À direita está uma demonstração de como esse código
ficaria após uma refatoração.

O script à esquerda muda a resolução do jogo ao trocar a string
`if (!((c->x[8] >> 0) & 1))` por outra com um flag a mais. Para isso,
alguém precisou descobrir que `x[8]` guarda o flag de configuração. Esse
conhecimento está num comentário do script e não no código em si, e, se
o jogo for atualizado, o endereço muda completamente e o patch deixa de
funcionar. Isso vai contra a proposta da refatoração.

### 4.4 O que piorou com a utilização das LLMs?

Além do acúmulo de remendos a cada iteração, o conhecimento sobre por
que o código funciona de determinada forma deixou de estar no próprio
projeto. As decisões ficam registradas na conversa com a IA, e não no
código ou na sua documentação. Mesmo com um repositório versionado, o
histórico apenas mostra o que foi alterado e raramente o raciocínio por
trás dessas mudanças.

Isso torna a análise e a melhoria do código mais difíceis para o usuário
comum. Assim, embora o processo de criação tenha sido muito
simplificado, a manutenção do código tornou-se muito mais complexa
devido à ausência de boas práticas de engenharia, como a documentação
das decisões.

## 5. Conclusão

Eu manteria a LLM realizando a maior parte do trabalho braçal de
comparação de bytes e de compilação, mas, em vez de focar na entrega
rápida de um produto, eu utilizaria a verificação humana para tornar o
código mais limpo e manter uma documentação atualizada do projeto, para
que outras pessoas possam criar seus próprios forks e desenvolver coisas
novas a partir do port original.

Até mesmo esse processo de apuração e melhoria do código pode ser feito
por meio de LLMs. É possível treinar uma LLM para jogar constantemente e
comparar o funcionamento do jogo, reportar bugs, tirar screenshots,
entre outras tarefas que agilizariam esse processo de melhoria do
código.

Dessa forma, além de utilizarmos as novas tecnologias do mercado a nosso
favor na criação de projetos, torna-se possível criar um conhecimento
próprio sobre os diferentes mecanismos de decompilação e recompilação de
código, aplicação de boas práticas de programação e de ensinamentos
importantes para futuros desenvolvedores. Isso também ampliaria o número
de projetos criados por pessoas que inicialmente não conseguiriam botar
suas ideias em prática por conta do código original.

## Referências

1.  NotRexed. [Hookshot](https://github.com/NotRexed/Hookshot)
    (repositório: `docs/design.md`, `docs/worklist.md`, `AGENTS.md`,
    `CONTRIBUTING.md`, `nx2/oot_patches.py`, `CHANGELOG.md`; commit
    `f9286c4`, 11/11/2026). GitHub. (Fonte primária.)
2.  zortos293. [Lichii](https://github.com/zortos293/Lichii)
    (repositório: `README.md`, `AGENTS.md`). GitHub, 2026. (Fonte
    primária.)
3.  [Nintendo's Ocarina of Time remake is three days old, and a
    Claude-built port already runs it on
    PC](https://madrobot.blog/2026/11/08/hookshotocarina-of-time-remake-pc-port-claude/).
    Mad Robot, 8 nov. 2026.
4.  [Hookshot Pulls the Switch 2 Ocarina of Time Remake Straight Onto
    Windows
    PCs](https://www.techeblog.com/hookshot-switch-2-ocarina-of-timeremake-pc-port-recomp/).
    TechEBlog, nov. 2026.
5.  [Zelda: Ocarina of Time Remake Gets a Native PC Port Through Static
    Recompilation](https://www.techpowerup.com/354592/zelda-ocarina-of-timeremake-gets-a-native-pc-port-through-static-recompilation).
    TechPowerUp, nov. 2026.
6.  [Zelda: Ocarina of Time Remake Runs on PC Emulator Just Hours After
    Nintendo Switch 2
    Release](https://www.keengamer.com/articles/news/zeldaocarina-of-time-remake-runs-on-pc-emulator-just-hours-after-nintendo-switch-2-release/).
    KeenGamer, nov. 2026.
7.  Mars7x. [Echora](https://github.com/Mars7x/echora). GitHub. (Fonte
    primária.)
8.  Zelda Reverse Engineering Team.
    [zeldaret/oot](https://github.com/zeldaret/oot). GitHub. (Fonte
    primária.)
9.  [Ship of
    Harkinian](https://en.wikipedia.org/wiki/Ship_of_Harkinian).
    Wikipédia.
10. LEWIS, Chris. [The Unexpected Effectiveness of One-Shot
    Decompilation with
    Claude](https://blog.chrislewis.au/the-unexpected-effectiveness-of-one-shot-decompilation-with-claude/).
    (Fonte primária.)
11. LEWIS, Chris. [Snowboard Kids 2 is 100%
    Decompiled](https://blog.chrislewis.au/snowboard-kids-2-is-100-decompiled/). 2026.
    (Fonte primária.)
12. TAN, H. et al. [LLM4Decompile: Decompiling Binary Code with Large
    Language Models](https://aclanthology.org/2024.emnlp-main.203/).
    EMNLP 2024. (Fonte primária, artigo acadêmico.)
13. [Unofficial PC ports of N64 games could be about to get a lot
    easier](https://videogameschronicle.com/news/unofficial-pc-ports-of-n64-games-could-be-about-to-get-a-lot-easier/).
    VGC.
14. FOWLER, Martin. *Refactoring: Improving the Design of Existing
    Code*. 2. ed. Addison-Wesley, 2018.
