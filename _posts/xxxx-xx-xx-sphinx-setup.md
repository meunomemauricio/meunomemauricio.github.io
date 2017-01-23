#### Configurando o Sphinx

Pra configurar o Sphinx, eu instalei ele em um Virtual Env e rodei o *Quick
Start* no diretório `docs` do repositório:

{% highlight terminal %}
$ pip install sphinx
$ mkdir docs
$ cd docs
$ sphinx-quickstart
{% endhighlight %}

O comando irá fazer várias perguntas sobre como iniciar o projeto de
documentação. Na segunda pergunta, sobre como será a divisão do diretório de
build, eu optei por separar o `source` e o `build` por achar que fica um pouco
mais organizado:

    You have two options for placing the build directory for Sphinx output.
    Either, you use a directory "_build" within the root path, or you separate
    "source" and "build" directories within the root path.
    > Separate source and build directories (y/n) [n]: y

Também é bom garantir que está instalando o `autodoc`, que gera documentação
automaticamente a partir das *docstrings* nos módulos Python:

    Please indicate if you want to use one of the following Sphinx extensions:
    > autodoc: automatically insert docstrings from modules (y/n) [n]: y

Pras outras opções, os valores padrão são suficientes.

No final do processo haverão os arquivos `Makefile` e `make.bat` e os
diretórios `source` e `build`.
