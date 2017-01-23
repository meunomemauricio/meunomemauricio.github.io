---
layout: post
comments: true
title: Publicação automática de documentação Sphinx no GitHub Pages
date: 2017-01-23 14:35:00
lang: pt
ref: sphinx_github
categories: Python
tags:
- Sphinx
- GitHub Pages
- Documentation
excerpt_separator: <!--more-->
---

O [**GitHub Pages**][github_pages] é um serviço do GitHub que fornece
hospedagem gratuita de páginas web a partir de repositórios Git, perfeito para
blogs pessoais e documentação de projetos.

O processo para criar uma página para documentar um projeto até que é simples:
basta criar um novo branch no repositório e incluir os arquivos `.html` nele.

Nesse post, irei mostrar como fazer para gerar documentação de um projeto
Python utilizando o [Sphinx][sphinx] e disponibilizar o resultado
automaticamente no GitHub pages.

<!--more-->

Para criar a página do GitHub Pages para um repositório já existente, basta
criar o branch `gh-pages` no repositório:

{% highlight terminal %}
$ git checkout --orphan gh-pages
$ git rm -rf .
$ echo "Hello World!" > index.html
$ git add .
$ git commit -m "Creating GitHub pages branch."
$ git push origin gh-pages
{% endhighlight %}

O branch é criado como `--orphan`, indicando que ele não conterá o histórico de
commits do `master`. Todo o conteúdo é removido do branch e é criado um único
arquivo `index.html` que será a página inicial. Por enquanto o único conteúdo é
`Hello World!` só para testar. Fazendo o commit e push, a página já deverá
estar disponível no endereço:

    https://<nome_de_usuario>.github.io/<nome_do_projeto>

Com isso funcionando, o processo pra incluir a documentação do Sphinx é
basicamente copiar o que for gerado no diretório `build` ou `_build`
(dependendo da configuração) para o root do branch `gh-pages`.

Pra fazer isso de maneira automática, podemos criar um novo `Makefile` no root
do projeto contendo:

{% highlight make %}
# Directories containing src files to generate documentation for GitHub page
GH_PAGES_SOURCES = docs

# Automatically builds and publishes the documentation
gh-pages:
	git checkout gh-pages
	rm -rf build _sources _static
	git checkout master $(GH_PAGES_SOURCES)
	git reset HEAD
	cd docs; make html
	mv -fv docs/build/html/* ./
	rm -rf $(GH_PAGES_SOURCES)
	git add -A
	git commit -m "Generated gh-pages for `git log master -1 --pretty=short --abbrev-commit`"
	git push origin gh-pages
	git checkout master
 
{% endhighlight %}

Linha a linha:

1. Checkout no branch `gh-pages`;
2. Remover diretórios já existentes (se não o `mv` depois pode falhar);
3. Copia os arquivos fonte de documentação do `master` (**Dependendo do projeto,
pode ser necessário incluir outros arquivos/diretórios na variável**
`GH_PAGES_SOURCES`);
4. Faz um reset para tirar os arquivos da *staging area*;
5. Faz um `cd` pro `docs` e faz o build do HTML;
6. Move os arquivos do diretório `docs/build/html` pro root do branch (**pode
ser necessário ajustar essa linha, dependendo das configurações do Sphinx**);
7. Remove os arquivos/diretórios com arquivos fonte;
8. Adiciona os novos arquivos na *staging area*;
9. Commita as alterações com uma mensagem automática apontando o SHA1
abreviado (do commit no `master`);
10. Faz o push para o branch remoto;
11. Checkout de volta no `master`;

> Aconselho a testar o novo Makefile sem as duas últimas linhas, para não fazer
push de besteira la pro repositório remoto.

Com isso, sempre que houver uma alteração na documentação no `master`, basta
commitar as alterações e depois rodar:

{% highlight terminal %}
$ make gh-pages
{% endhighlight %}

#### Atribuições

A receitinha do Makefile para gerar e publicar a documentação automaticamente
foi adaptada a partir deste post no [Nikhil's Blog][nikhil]

[github_pages]: https://pages.github.com/
[sphinx]: http://www.sphinx-doc.org/en/stable/
[nikhil]: http://nikhilism.com/post/2012/automatic-github-pages-generation-from/
