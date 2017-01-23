Isso não é necessário, mas eu também gosto muito do visual do [Read The
Docs][rtd] e é possível utilizar um theme dele no Sphinx. Basta instalar o
pacote:

{% highlight terminal %}
$ pip install sphinx_rtd_theme
{% endhighlight %}

e incluir no arquivo `docs/source/conf.py`:

{% highlight python %}
# -- Read The Docs theme --------------------------------------------------
import sphinx_rtd_theme

html_theme = "sphinx_rtd_theme"
html_theme_path = [sphinx_rtd_theme.get_html_theme_path()]
{% endhighlight %}

Com isso, o Sphinx ta configurado. Basta criar os arquivos reST no diretório
`source` e gerar a documentação com `make html` no diretório `docs`. A
documentação estará disponível no diretório `build`.

