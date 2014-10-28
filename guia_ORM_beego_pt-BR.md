Este é um guia informal baseado no que é entendido como manuseo do ORM beego.Devo declarar que não sou nenhum especialista no tema, estou aprendendo a linguagem `GO`, e quero compartilhar alguns conhecimentos adquiridos.
***
Se puder fornecer quaisquer comentários ou explicações que ajude a melhorar esse guia seja bem vindo.
***
Haverá alguns conceitos que se devem ter claros inicialmente para entendimento, como o que é um ORM, o que é SQL, todos os termos relacionados com Bases de dados e todo manuseio da linguagem `GO`. Estas explicações não serão tratadas aqui, já que se podem encontrar muitas e muito boas explicações na internet, coisa que sugiro que faça antes de aprender manusear este ORM.
***
Para utilizar este ORM, gostaria de partir de um caso hipotético onde precisaria do uso e implementação de uma base de dados com suas respectivas tabelas, que logo utilizaremos para realizar algumas consultas e modificações, na medida que vamos precisando.

Vamos trabalhar com uma tabela `artigos` e uma tabela `iventario`.

***
Vamos criar o arquivo main.go, com os parametros necessários para chamar a base de dados e dar os parâmetros necessários.

```go
package main

  import (
  . "fmt"
  "github.com/astaxie/beego/orm"
  _ "github.com/go-sql-driver/mysql"
  )

  func init() {
  orm.RegisterDriver("mysql", orm.DR_MySQL)//registrando o driver para mysql
  orm.RegisterDataBase("default", "mysql", "root:senha_de_acceso@/faturacao?charset=utf8")
  }

  func main() {
  o := orm.NewOrm()
  o.Using("default")
  //ativado para ver log no console
  orm.Debug = true

  }
```

Depois que a importação do motor correspondente da base de dados que vai usar, neste caso mysql, vemos que a função init(), registramos o driver e a base de dados como "default". Neste caso a base de dados "faturacao" é a que vamos utilizar por "padrão".
***
Este ORM, suporta três bases de dados:
```go
  import (
    _ "github.com/go-sql-driver/mysql"
    _ "github.com/lib/pq"
    _ "github.com/mattn/go-sqlite3"
  )
```

E para o registro podemos passar na função orm.RegisterDriver();
```go
  orm.DR_MySQL
  orm.DR_Sqlite
  orm.DR_Postgres
```

Ao registrar a base de dados podemos definir 2 parâmetros para que são opcionais, SetMaxIdleConns e SetMaxOpenConns, uma das formas de fazer é:
```go
  func init() {
  maxIdle := 30
  maxConn := 30
  orm.RegisterDriver("mysql", orm.DR_MySQL)
  orm.RegisterDataBase("default", "mysql", "root:senha_de_acceso@/faturacao?charset=utf8", maxIdle,     maxConn)
  }
```

No caso de somente usar a base de dados por padrão não é necessário usar
```go
`o.Using("default")`
```
***

##REGISTRANDO O MODELO

Se quiser usar o ORM beego, em todo seu contexto, então precisamos obrigatóriamente registrar o modelo, para isto o recomendável é usar apenas um arquivo models.go, que irá definir a estrutura da tabela mediante uso dos tipos struct.

Para que o ORM, não cria as tabelas, devemos criar um objeto tipo struct, com o nome da tabela e o nome dos campos da tabela serão os campos da estrutura, algumas tabelas podem em um caso real necessitar muito mais campos especifícos, mas para o exemplo, usaremos os dados mais importantes na minha opinião.
```go
  package main

  import (
  "github.com/astaxie/beego/orm"
  )

  type Artigos struct {
  Id          int
  Descricao string
  }

  type Iventario struct {
  Id        int
  Artigos *Artigos `orm:"rel(fk)"`
  VlrCusto  float64
  Entrada   int
  Saida    int
  }

  func init() {
  orm.RegisterModel(new(Artigos), new(Iventario))
  }
```

O código acima vamos salvar no arquivo models.go. Neste código podemos observar que a estrutura Iventário tem "Artigos *Artigos e se define `orm:"rel(fk)"`, com isso estamos dizendo que este campo é um campo de relação com a chave estrangeira da tabela artigos.

Ao registrar o modelo com orm.RegisterModel(new(Nome da tabela)), isto vai servir para criar a tabela e note que os nomes das tabelas serão criados mudando sua letra inicial de maiúscula para minúscula, o mesmo para o nome dos campos.

Ao registrar o modelo, podemos querer utilizar um prefixo para o nome da tabela, para o qual vamos usar:

orm.RegisterModelWithPrefix("prefix_", new(Inventario))

O nome da tabela será criado como prefix_iventario.
***
Para criar as tabelas pode fazer uso do código diretamente ou usar comandos no terminal; Se deve ter em mente que faze-lo por código, então deve desativar a criação, caso contrário cada vez qeu o código for executado, será criado e sobreescreverá as tabelasno caso de ter o parâmetro `force := false`, não se sobreescreverá mas vai fazer verificação de código se a tabela existe ou não desnecessariamente.

Se quiser o código, usamos:

```go
  func main() {
  o := orm.NewOrm()
  orm.Debug = true
  o.Using("default")

  //Cria as tabelas
  name := "default"
  // Apaga tabelas e as cria novamente.
  force := true
  // Print log.
  verbose := true
  // Error.
  err := orm.RunSyncdb(name, force, verbose)
  if err != nil {
    Println(err)
  }
```

Se quisermos fazer por terminal:

```go
  func main() {
  o := orm.NewOrm()
  orm.Debug = true
  o.Using("default")
  //ativa comandos no terminal
  orm.RunCommand()
  }
```
Então digitamos no console:

`:~# go build main.go`
`:~# ./relacoes orm syncdb -db="default" -force=1 -v`
***

##OPERAÇÕES
Para os seguintes exemplos pode optar por inserir os dados nas tabelas de forma manual pelo console mysql, ou usando phpmyadmin, se preferir graficamente. Depis explico como se pode inserir os dados mediante código.

Podemos usar Inser,Update,Read e Delete, diretamente se conhecermos a chave primária:
```go
  o := orm.NewOrm()
  artigos := new(Artigos)
  artigos.Descricao = "Artigo10"
  o.Insert(artigos)//insere "Artigo10" na tabela "artigos"

  artigos.Id = 10
  o.Delete(artigos)//Apaga registro com o "Id=10" da tabela "artigos"
```

Os dois seguintes trechos de códigos encontram o registro como `Id` correspondente.
```go
  artigo := new(Artigos)
  artigo.Id = 13
  err = o.Read(artigo)

  if err == orm.ErrNoRows {
    Println("No result found.")
  } else if err == orm.ErrMissPK {
    Println("No primary key found.")
  } else {
    Println(artigo.Descripcion)
  }
```
Este código fará o mesmo:
```go
  artigo := Artigos{Id: 13}
  err := o.Read(&artigo1)

  if err == orm.ErrNoRows {
    Println("No result found.")
  } else if err == orm.ErrMissPK {
    Println("No primary key found.")
  } else {
    Println(artigo.Descripcion)
  }
```
`Read` sempre usa a chave primária por padrão, se quiser ler um dado buscando outra columa, deve especificar:
```
  artigos := Artigo{Descricao: "Artigo10"}
  err := o.Read(&artigos, "Descricao")

  if err == orm.ErrNoRows {
    Println("No result found.")
  } else if err == orm.ErrMissPK {
    Println("No primary key found.")
  } else {
    Println(artigos.Id)
  }
```
`ReadOrCreate` procuram combinar de acordo com o parâmetro que nós lhe demos, se prova em contrário, criar o registro.
```go
  artigos := Artigos{Descricao: "Artigo2"}
  if created, id, err := o.ReadOrCreate(&artigos, "Descricao"); err == nil {
    if created {
      Println("novo objeto inserido. Id:", id)
    } else {
      Println("O objeto é. Id:", id)
    }
  }
```
Podemos com `Insert`, inserir um dado:
```go
  var art Artigos
  art.Descricao = "artigo15"

  id, err := o.Insert(&art)
  if err == nil {
    Println(id)//o campo id é chave primária e autoincremental
  }
```
Ainda Inserir múltiplos dados podemos usar:
```go
  arts := []Artigos{
    {Descricao: "artigo_za"},
    {Descricao: "artigo_zb"},
    {Descricao: "artigo_zc"},
  }
  successNums, err := o.InsertMulti(1, arts)
  Println(successNums, err)
```

Se o número da função `InsertMulti(1,arts)`é maior que 1, a consulta de inserção de dados insere todos ao mesmo tempo, se não insere um por um.
***
para atualizar registros:
```go
  artigos := Artigos{Id: 1}
  if o.Read(&artigos) == nil {
  artigos.Descricao = "Nova_descricao"
      if num, err := o.Update(&artigos); err == nil {
          Println(num)
        }
  }

```
E para apagar um registro podemos usar este código:
```go
  if num, err := o.Delete(&Artigos{Id: 58}); err == nil {
    Println(num)
  }
```

##CONSULTAS AVANÇADAS

-. Buscando registros de uma tabela relacionada.
```go
  iventario := &Iventario{}
  o.QueryTable("iventario").Filter("artigos__id", 2).RelatedSel().One(iventario)
  Println(iventario.Artigos.Descricao)
```
Podemos encontrar o dado `Descricao`, do artigo com `Id`2 que se encontra na tabela `iventario`, uma vez que esta tabela se encontra relacionada, pelo parâmetro `orm:rel(pk)`, que se colocou no modelo da estrutura de `Iventarios`.

-. Buscando registros na tabela Iventario, com base no Id de artigos1
```go
  var inven Iventario
  err := o.QueryTable("iventario").Filter("artigos__id", 2).One(&inven)
  if err == nil {
    Println(inven.Entrada)
  }
```
Da mesma maneira podemo usar vários operadores, dentro das expressões da consulta:

Os operadores compatíveis:

exato / iexact igual a
contém / icontains contém
GT / GTE mario que / mario que ou igual a
LT / LTE menor de / menor de ou igual a
StartsWith / istartswith começa com
endswith / iendswith termina com
en
isnull

Os operadores que começam com i ignorar caso.

o uso destes operadores determinal a condição da consulta:
```go
  var maps []orm.Params
  num, err := o.QueryTable("iventario").Filter("vlr_custo__gte", 100).Values(&maps)
  if err == nil {
    Printf("Result Nums: %d\n", num)
    for _, m := range maps {
      Println(m["Id"])
    }
  }
```
O resultado será uma lista dos Id de iventario que sejam >= a 100

Você pode usar duas regras de filtragem, para ter e excluir, Para usar um `and`:
```go
  var maps []orm.Params
  num, err := o.QueryTable("iventario").Filter("vlr_custo__gte", 100).Filter("entrada__gte",     40).Values  (&maps)
  if err == nil {
    Printf("Result Nums: %d\n", num)
    for _, m := range maps {
      Println(m["Id"])
    }
  }
```
retornará valores de Id, maiores ou iguais a 100, na Valr_Custo `y` entrada >= 40

também pode escrever:
```go
  var maps []orm.Params

  qs := o.QueryTable("iventario")
  num, err := qs.Filter("vlr_custo__gte", 100).Filter("entrada__gte", 40).Values(&maps)

  if err == nil {
    Printf("Result Nums: %d\n", num)
    for _, m := range maps {
      Println(m["Id"])
    }
  }
```
Para obter valores na ordem crescente ou decrescente podemos usar:
decressente "-vlr_custo" ou cresente "vlr_custo"
```go
  var maps []orm.Params
  qs := o.QueryTable("iventario")
  num, err := qs.OrderBy("-vlr_costo").Values(&maps)

  if err == nil {
    Printf("Result Nums: %d\n", num)
    for _, m := range maps {
      Println(m["Id"], m["VlrCosto"])
    }
  }
```
Para fazer consultas relacionais e fazer um join:
```go
  var iventario []*Iventario
  qs := o.QueryTable("iventario")
  num, err := qs.OrderBy("vlr_custo").RelatedSel("artigos").All(&iventario)

  if err == nil {
    Printf("Result Nums: %d\n", num)
    for _, v := range iventario {
      Println(v.Id, v.Artigos.Id, v.Artigos.Descricao, v.VlrCusto)
    }
  }
```
com o código anterior fazeos uma consulta na tabela iventario, ordenado decrescente pelo "vlr_custo" e relacionaremos a tabela artigos para pegar os dados do artigo.

Com o parâmetro "All", podemos obter somente os campso que queremos:
```go
num, err := qs.OrderBy("vlr_custo").RelatedSel("artigos").All(&iventario, "id", "vlr_custo")
```
Pode-se observar também que nos diferentes códigos exibidos anteriormente, vão encontrar uma saída; .One(), .All(), .Values(), entre outros, pode-se provar caso necessite a saída e para alguns a variável que tem resultado é diferente. Para mais informações sobre as saídas por favor revisar `http://beego.me/docs/mvc/model/query.md`.

SQL Raw para consultar

Se vamos usar sql raw, não precisamos registrar o modelo.

Esta consulta executará o update:
```go
  res, err := o.Raw("UPDATE artigos SET descricao = ? WHERE id=?", "artigo_a",1).Exec()
  if err == nil {
    num, _ := res.RowsAffected()
    Println("mysql row affected nums: ", num)
  }
```
QueryRow: retorna um registro
```go
  var artigo Artigos
  err := o.Raw("SELECT id, descripcion FROM artigos WHERE id = ?", 1).QueryRow(&artigo)
  if err == nil {
    Println(artigo)
  }
```
QueryRows: retorna uma fatia dos dados
```go
  var artigos []Artigos
  num, err := o.Raw("SELECT id, descripcion FROM artigos WHERE id>?", 1).QueryRows(&artigos)
  if err == nil {
    Println(num, err, artigos)
  }
```
SetArgs:podemos usar a mesma consulta com diferentes parâmetros
```go
  o := orm.NewOrm()
  r := o.Raw("UPDATE artigos SET descricao = ? WHERE id=?")
  res, err := r.SetArgs("art_5", 5).Exec()
  res, err = r.SetArgs("art_6", 6).Exec()
```
Obtendo dados:
```go
  o := orm.NewOrm()
  var lists []orm.ParamsList
  num, err := o.Raw("SELECT descricao FROM artigos WHERE id > ?", 3).ValuesList(&lists)
  if err == nil && num > 0 {
    Println(lists)
  }
```
saída [[art_4] [art_5] [art_6] [articulo_7] [articulo_8]]
```go
  con Println(lists[0][0])
```
saída art_4
```go
  o := orm.NewOrm()
  var list orm.ParamsList
  num, err := o.Raw("SELECT descricao FROM artigos WHERE id > ?", 3).ValuesFlat(&list)
  if err == nil && num > 0 {
    Println(list)
  }
```
saída [art_4 art_5 art_6 articulo_7 articulo_8]
```go
con Println(list[3])
```
saída articulo_7
```go
  o := orm.NewOrm()
  res := make(orm.Params)
  num, err := o.Raw("SELECT id,descricao FROM artigos").RowsToMap(&res, "id", "descricao")
  if err == nil && num > 0 {
    Println(res)
  }
```
map[6:art_6 7:artigo_7 8:artigo_8 1:artigo_a 2:art_2 3:art_3 4:art_4 5:art_5]
```go
con Print(res["5"])
```
saída art_5

Como podemos notar, os valores de resultado retornados pela consulta SQL Raw são string.

Pode usar Prepare, para realizar a consulta uma vez e logo várias execuções:
```go
  o := orm.NewOrm()
  p, err := o.Raw("UPDATE artigos SET descricao = ? WHERE id = ?").Prepare()
  res, err := p.Exec("artigo_A", "1")
  res, err = p.Exec("artigo_D", "4")

  p.Close()//não esquecer de fechar o prepare
```
***





##NOTAS ADICIONAIS:
-. Manual original [BeegoORM](http://beego.me/docs/mvc/model/overview.md)
-. Documentação [MySQL](http://dev.mysql.com/doc/)
-. sobre o uso de [Paginacion OFFSET]( http://www.cristalab.com/tutoriales/optimizar-consultas-sql-para-paginar-resultados-en-mysql-c92972l/)
-.

