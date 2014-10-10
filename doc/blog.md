jelastic python
===============

Essa semana me foi dado a tarefa de testar o suporte a python no
Jelastic da locaweb. Nada muito rebuscado, apenas criar uma aplicação
web e relatar a experiência. Será bem interessanto pois será o meu
primeiro contato com o Jelastic.

Para quem ainda não conhece, o Jelastic é um PaaS (do inglês
*Plataform As A Service*) que permite criar e gerenciar os serviços
necessários para publicar seus projetos Php, Java, Ruby e agora
Python. Além da aplicação em si, também é possível gerenciar bancos
dados, cache e balanceadores de carga. Tudo isso com alguns cliques de
botão. Propaganda a parte, vamos ao projeto.

Minha ideia foi criar uma *API REST* para o famoso e antigo jogo da
velha. É um jogo com regras simples e imagino que não necessita
introdução, mas caso alguém tenha esquecido de como funciona o artigo
da Wikipedia [1] é uma boa referência.

Vou começar criando um projeto no github [2]. Não é necessário visto
que existem outros métodos de publicação mas simplifica
consideravelmente todo o processo. Todo o código deste artigo está
publicado neste repositório e pode ser usado como referência.

    $ git clone https://github.com/dgvncsz0f/tic-tac-toe.git
    $ cd tic-tac-toe

Neste projeto específico vou usar *flask*, que é um *framework* leve
para criar serviços HTTP. Vou deixá-lo no repositório, tornando-o
auto-contido, ou seja, o projeto já contém todas as dependências que
necessita para ser executado:

    $ pip install --target vendor flask
    $ pip install --target vendor redis
    $ git add vendor
    $ git commit -m 'adiciona a lib flask e redis no diretório vendor'

Feito isso é hora de criar a aplicação. Agora vou apenas definir o
esqueleto, sem adicionar nenhuma funcionalidade:

    $ cat <<ENDL >application
    #!/usr/bin/python
     
    import os
    import sys
    sys.path.append(os.path.join(os.path.dirname(os.path.realpath(__file__)), "vendor"))
     
    from flask import Flask
     
    application = Flask(__name__)
     
    @application.route("/")
    def root ():
        return "okie dokie", "text/plain; utf-8"
     
    if __name__ == "__main__":
        application.run()
    ENDL

    $ git add application
    $ git commit -m 'arquivo application com rota / retornando uma string constante'

É importante entender este código antes de seguir adiante. A primeira
parte deste código adiciona o diretório *vendor* no *PATH* do
python. Assim, o interpretador será capaz de encontrar o *flask* e o
*redis* que instalamos na raiz do projeto:

    sys.path.append(os.path.join(os.path.dirname(os.path.realpath(__file__)), "vendor"))

A partir daí, já podemos testar nosso código. Basta executar o módulo
que um servidor *HTTP* é iniciado e começa a servir as requisições.

    $ python2 application
    * Running on http://127.0.0.1:5000/

    $ curl http://127.0.0.1:5000/
    okie dokie

Hora de publicar este código no Jelastic. Em primeiro lugar criamos o
ambiente seguido pelo projeto. O ambiente permite o usuário escolher a
versão do python que se deseja usar (neste caso 2.7) e o projeto
determina como o código é publicado. Neste caso específico o próprio
Jelastic continuamente monitora o github e já faz a publicação sempre
que houver uma mudança no código, convenientemente.

Também vou precisar de um banco de dados para manter estado dos jogos
em andamento, que neste caso será o redis.

    [[ screenshot ]]

Tudo muito simples e rápido até agora vamos finalmente criar o jogo. O
servidor será responsável por manter o estado dos jogos em
andamento. Para simplificar as coisas vamos assumir um cenário
perfeito no qual os jogadores são 100% honestos.

O módulo mais importante é o `tictactoe.game`. Este módulo contém o
estado do jogo e métodos importantes como verificar se há algum
ganhador ou o próximo jogador. Há outros dois módulos importantes, o
`tictactoe.pp` que cuida de imprimir o estado do jogo num formato
fácil de interpretar e o módulo `tictactoe.db` que é a interface para
o banco de dados.

O estado do jogo é mantido como um `array` de 9 posições. Cada item
desta estrutura pode assumir três posições: `'x'`, `'o`' ou `None` que
significam respectivamente jogador 1, jogador 2 ou área em aberto:

    def __init__ (self, state=None):
        if state is None:
            state = [None for _ in range(9)]
        self.state = state
    
Com isso em mãos podemos implementar um método que verifica linhas e
colunas e retorna o jogador ganhador, caso haja algum:

    def check_rows_n_cols_ (self):
        for i in range(3):
            j = i * 3
            data = set(self.state[j : j + 3])
            if self.has_winner_(data):
                return data.pop()

            data = set([self.state[i], self.state[3 + i], self.state[6 + i]])
            if self.has_winner_(data):
                return data.pop()

Analogamente, a seguinte função verifica se há algum ganhador em
alguma das duas diagonais:

    def check_diagonals_ (self):
        data = set([self.state[0], self.state[4], self.state[8]])
        if self.has_winner_(data):
            return data.pop()

        data = set([self.state[2], self.state[4], self.state[6]])
        if self.has_winner_(data):
            return data.pop()

Ficou faltando o método auxiliar `has_winner_`, descrito a seguir:

    def has_winner_ (self, s):
        return len(s) == 1 and s != set([None])

Com isso temos um modelo básico do jogo da velha. Foram omitidas
algumas linhas principalmente relacionadas a manter a invariante do
jogo, aqueles mais curiosos vão encontrar o código completo, bem como
uma conjunto básico de testes no github.

É chegada a hora de implementar a *API REST*. Vamos definir 4
recursos, detalhados a seguir:

    * /new: registra um novo jogo;
    
    * /play/<game-id>: executa uma jogada. O próprio servidor seleciona e
      alterna o jogador automáticamente;

    * /show/<game-id>: mostra o estado de um determinado;

    * /show/<game-id>/winner: mostra o ganhador do um determinado jogo;