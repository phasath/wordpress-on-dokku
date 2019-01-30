# Deploy de Wordpress no Dokku

## TODO
- Criar versão em inglês da documentação

## Sobre o template

Esse template foi criado para realizar o deploy de um site WordPress do Dokku. Foi testado no dia 11 de dezembro de 2018.
O template utiliza o `buildpack` oficial do [Heroku para PHP](https://github.com/heroku/heroku-buildpack-php) com algumas configurações específicas.
Esse template também foi desenvolvido para suportar **apenas** um Wordpress, e não multi-sites.

O NGINX desse repositório também foi modificado de forma a otimizar o uso do Wordpress, cacheamento e afins, além de incrementar a segurança do mesmo.

> Esse template foi criado analisando o código fonte e funcionamento do Dokku, do Buildpack de PHP do Heroku e do Wordpress. Caso haja algum problema, abra uma [issue](https://github.com/phasath/wordpress-on-dokku/issues/new) para que possamos analisar.

## Requerimentos

- Servidor com Dokku pré-configurado
- [Plugin do MariaDB para o Dokku](https://github.com/dokku/dokku-mariadb)
- [Composer](https://getcomposer.org) na sua máquina de desenvolvimento
- Fork do Git para que você possa fazer as suas alterações conforme o seu necessário.

> Usaremos como domínio global, nesse exemplo, o nome **exemplo.com**.

## Procedimentos

### Migrando o Wordpress

Durante o tutorial, alguns passos serão necessários em caso de migração. Conforme o tutorial, será avisado o momento onde realizar cada passo. Para começar, já realize o passo 1.

1. Realizar o **dump** do banco de dados anterior;
2. Restaurar o **dump** criado no passo 1 no banco criado;
3. Acessar o banco e rodas as seguintes queries, lembrando de alterar o prefixo das tabelas se não for o padrão `wp_`:
```
UPDATE wp_options SET option_value = replace(option_value, 'http://www.oldurl', 'http://www.newurl') WHERE option_name = 'home' OR option_name = 'siteurl';

UPDATE wp_posts SET guid = replace(guid, 'http://www.oldurl','http://www.newurl');

UPDATE wp_posts SET post_content = replace(post_content, 'http://www.oldurl', 'http://www.newurl');

UPDATE wp_postmeta SET meta_value = replace(meta_value,'http://www.oldurl','http://www.newurl');
```
4. Copiar o conteúdo das pastas **uploads**, **plugins** e **themes** se usar, para as pastas persistentes que criadas;
5. Acessar o Painel Administrativo do Wordpress e reativar os plugins;
6. Por algum motivo desconhecido, é necessário alterar a forma como link permanente é feito. Para tal, mude para outro formato, salve. Volte para o seu formato antigo e salve novamente.


### Criar um app no dokku

Neste exemplo, iremos usar um aplicação chamada `wordpress-teste`

```
$ dokku apps:create wordpress-teste
```

### Configurar a aplicação

No nosso template, as configurações do `wp-config.php` do Wordpress são obtidas através de variáveis de ambiente. 
Precisamos setar as seguintes variáveis:

- AUTH_KEY
- SECURE_AUTH_KEY
- LOGGED_IN_KEY
- NONCE_KEY
- AUTH_SALT
- SECURE_AUTH_SALT
- LOGGED_IN_SALT
- NONCE_SALT
- TABLE_PREFIX
- DEBUG_FLAG

Com exceção das duas últimas, todas são chaves de autenticação únicas e sais usados pelo Wordpress para garantir segurança para os usuários ao criptografar as senhas e etc.

A variável `TABLE_PREFIX` é utilizada caso o prefixo das tabelas do Wordpress não seja o padrão `wp_`. Não esqueça de colocar o **_** (underline) no nome da tabela.

Já a variável `DEBUG_FLAG` ativa o modo debug do WordPress permitindo que você possa debuggar e encontrar os erros. 

Quaisquer outras alterações à niveis de `wp-config.php` do Wordpress devem ser feitas no arquivo `wp-config.php` dentro do diretório `wordpress-config`.

Para setar essas variáveis, usaremos o seguinte comando. Somente altere a parte do `TABLE_PREFIX` se o prefixo das suas tabelas não forem **wp_** e a `DEBUG_FLAG` caso queira debuggar.

```
$ dokku config:set wordpress-teste AUTH_KEY=$(</dev/urandom tr -dc 'A-Za-z0-9d#$%&*+,-./:;<=>?@[\]^_{|}~' | head -c 64  ; echo) SECURE_AUTH_KEY=$(</dev/urandom tr -dc 'A-Za-z0-9d#$%&*+,-./:;<=>?@[\]^_{|}~' | head -c 64  ; echo) LOGGED_IN_KEY=$(</dev/urandom tr -dc 'A-Za-z0-9d#$%&*+,-./:;<=>?@[\]^_{|}~' | head -c 64  ; echo) NONCE_KEY=$(</dev/urandom tr -dc 'A-Za-z0-9d#$%&*+,-./:;<=>?@[\]^_{|}~' | head -c 64  ; echo) AUTH_SALT=$(</dev/urandom tr -dc 'A-Za-z0-9d#$%&*+,-./:;<=>?@[\]^_{|}~' | head -c 64  ; echo) SECURE_AUTH_SALT=$(</dev/urandom tr -dc 'A-Za-z0-9d#$%&*+,-./:;<=>?@[\]^_{|}~' | head -c 64  ; echo) LOGGED_IN_SALT=$(</dev/urandom tr -dc 'A-Za-z0-9d#$%&*+,-./:;<=>?@[\]^_{|}~' | head -c 64  ; echo) NONCE_SALT=$(</dev/urandom tr -dc 'A-Za-z0-9d#$%&*+,-./:;<=>?@[\]^_{|}~' | head -c 64  ; echo) TABLE_PREFIX=wp_ DEBUG_FLAG=false
```

> O comando `</dev/urandom tr -dc 'A-Za-z0-9d#$%&*+,-./:;<=>?@[\]^_{|}~' | head -c 64  ; echo` gera uma chave de 64 caractéres aleatórios com base na OWASP. Foram removidos alguns caracteres para evitar problemas com o dokku. 

### Criação e conexão com o banco

Criaremos um banco usando o [plugin do MariaDB para Dokku](https://github.com/dokku/dokku-mariadb):

```
$ dokku mariadb:create wordpress-teste-db
```

Agora, iremos linkar ele com nossa aplicação, disponibilizando a url de acesso na nossa aplicação:

```
$ dokku mariadb:link wordpress-teste-db wordpress-teste
```

Feito isso, teremos adicionado uma variável chamada `DATABASE_URL` nas variáveis de configuração da nossa aplicação. 

Caso você queira ter acesso à aplicação, pode usar o comando para mapear uma porta no **host** para o banco dentro de um **container docker**.

```
$ dokku mariadb:expose wordpress-teste-db
```

> Caso você esteja realizando uma migração, execute agora o passo 2 e 3 da subseção Migrando o Wordpress

Para verificar se tudo está configurado direitinho, você pode usar o comando:

 ```
$ dokku config wordpress-teste
```

### Montando os volumes permanentes

O container do Dokku é efêmero, isto quer dizer que, sempre que houver um novo build ou deploy, tudo voltará para o formato de como se encontra o repositório enviado. Por isso, geralmente é desejável que os arquivos de **uploads** e **plugins** sejam permanentes. Se você for utilizar um tema do WordPress que não esteja no seu repositório, deveria criar um volume permanente tmabém para a pasta de temas. 

Neste exemplo, criaremos as pastas uploads e plugins no **host** no [diretório recomendado pelo dokku](https://github.com/dokku/dokku/blob/master/docs/advanced-usage/persistent-storage.md#persistent-storage):

Temas dentro do repositório:
```
$ mkdir -p /var/lib/dokku/data/storage/wordpress-test/{uploads,plugins} 
```

Usando um tema do wordpress:
```
$ mkdir -p /var/lib/dokku/data/storage/wordpress-test/{uploads,plugins,themes} 
```

E agora, linkaremos nossa aplicação com esses diretórios. Novamente, usando um tema no repositório:
```
$ dokku storage:mount wordpress-test /var/lib/dokku/data/storage/wordpress-test/uploads:/app/wp-content/uploads
$ dokku storage:mount wordpress-test /var/lib/dokku/data/storage/wordpress-test/plugins:/app/wp-content/plugins
```

Usando um tema do Wordpress, adicione esse mapeamento também:
```
$ dokku storage:mount wordpress-test /var/lib/dokku/data/storage/wordpress-test/themes:/app/wp-content/themes
```

> Caso você esteja realizando uma migração, execute agora o passo 4 da subseção Migrando o Wordpress

Em termos de configuração, está tudo pronto. Precisamos apenas configurar os detalhes de nosso wordpress.


### No template

A única alteração a ser feita aqui é colocar dentro da pasta `wordpress-config` toda e quaisquer coisas que não sejam default do wordpress. Por exemplo:
Se eu quisesse que o tema fosse sincronizado, dentro da pasta `wordpress-config`, deveria criar uma pasta chamada `wp-content` e dentro dela, a past a `themes`.

> Todas as pastas devem seguir o modelo do Wordpress para que funcione adequadamente. 

### Atualização do Composer

Você pode atualizar o `composer.json` para condizer com sua aplicação sem alterar a parte de `scripts` e `extra`.

Uma vez realizado, utilize o comando

```
$ composer install
```

Ou, caso já tenha instalado e precise apenas atualizar alguma modificação:

```
$ composer update
```

### Enviar para o dokku

Obviamente, comit e push das suas mudanças, lembrando de escrever commits atômicos (um conjunto de mudanças com um único propósito) e outras [boas práticas de Git](https://medium.com/@rafael.oliveira/como-escrever-boas-mensagens-de-commit-9f8fe852155a)

Agora, é preciso apenas enviar para o Dokku. Neste exemplo, vamos supor que o seu Dokku esteja instalado numa máquina chamada `meu-dokku`:
```
git remote add dokku dokku@meu-dokku:wordpress-test
```

E por fim:
```
git push dokku master:master
```

> Caso você esteja realizando uma miração, execute os passos 5 e 6 agora.

### Usando HTTPS (LetsEncrypt)


Para usar HTTPS, usaremos o certificado do **LetsEncrypt**, que já existe um [plugin do LetsEncrypt no Dokku](https://github.com/dokku/dokku-letsencrypt) que cria para nós automaticamente.

Dado que o plugin esteja instalado, use o comando:
```
$ dokku letsencrypt wordpress-test
```

Talvez seja necessário alterar no banco o endereço do site de HTTP para HTTPS. Isso pode ser realizado de duas formas:
- Via Painel ADM do Wordpress ANTES de usar o LetsEncrypt.
- Via Queries no Banco de Dados.

Caso opte pela segunda opção:
```
UPDATE wp_options SET option_value = replace(option_value, 'http://www.oldurl', 'http://www.newurl') WHERE option_name = 'home' OR option_name = 'siteurl';

UPDATE wp_posts SET guid = replace(guid, 'http://www.oldurl','http://www.newurl');

UPDATE wp_posts SET post_content = replace(post_content, 'http://www.oldurl', 'http://www.newurl');

UPDATE wp_postmeta SET meta_value = replace(meta_value,'http://www.oldurl','http://www.newurl');
```

#### Fontes
[Lista de Caractéres Especiais da OWASP](https://www.owasp.org/index.php/Password_special_characters)