Este es una guia informal basada en lo que he entendido del manejo de la ORM beego. Debo de aclarar que no soy ningun experto en el Tema, estoy tratando de aprender el Lenguaje `GO`,  y he querido compartir algunos conocimientos adquiridos.
***
Si puedes aportar algun comentario o explicacion que ayude a mejorar esta guia bienvenido sea.
***
Habra algunos conceptos se deden tener claros de antemano o por lo menos entenderlos, como que es una ORM, que es SQL, todos los terminos relacionados con las Bases de datos y por su puesto el manejo del lenguaje `GO`.  Estas explicaciones no se trataran aqui, ya que se pueden encontrar muchas y muy buenas explicaciones en la red, cosa que sugiero se haga antes de tratar de aprender el manejo de esta ORM.
***
Para utilizar esta ORM, me gustaria partir de un caso hipotetico donde se necesitaria el uso e implementacion de una base de datos con sus respectivas tablas, las que luego utilizaremos para realizar algunas consultas y modificaciones, segun se vayan necesitando.

Vamos a trabajar con una trabla `articulos` y una tabla `inventario`.

***
Vamos a crear el archivo main.go, con los parametros necesarios para llamar a la base de datos y dar los parametros que se necesiten.

```go
package main

	import (
	. "fmt"
	"github.com/astaxie/beego/orm"
	_ "github.com/go-sql-driver/mysql"
	)

	func init() {
	orm.RegisterDriver("mysql", orm.DR_MySQL)//registrando el driver para mysql
	orm.RegisterDataBase("default", "mysql", "root:clave_de_acceso@/facturacion?charset=utf8")
	}

	func main() {
	o := orm.NewOrm()
	o.Using("default")
	//activado para ver log en la consola
	orm.Debug = true

	}
```

Luego de la importacion correspondiente al motor de bases de datos a usar, en este caso mysql, vemos que en la funcion init(), registramos el driver y la base de datos con "default".  En este caso la base de datos "facturacion", es la que se va utilizar por "default".
***
Esta ORM, soporta tres bases de datos:
```go
	import (
    _ "github.com/go-sql-driver/mysql"
    _ "github.com/lib/pq"
    _ "github.com/mattn/go-sqlite3"
	)
```

Y para el registro podemos pasarle a la funcion orm.RegisterDriver():
```go
	orm.DR_MySQL
	orm.DR_Sqlite
	orm.DR_Postgres
```

Al registrar la base de datos podemos definir 2 parametros mas que son opcionales, SetMaxIdleConns y SetMaxOpenConns, una de las formas de hacerlo es:
```go
	func init() {
	maxIdle := 30
	maxConn := 30
	orm.RegisterDriver("mysql", orm.DR_MySQL)
	orm.RegisterDataBase("default", "mysql", "root:clave_de_acceso@/facturacion?charset=utf8", maxIdle, 		maxConn)
	}
```

En el caso de que solamente vayamos a usar la base de datos por defecto no necesitamos usar 
```go
`o.Using("default")`
```
***

##REGISTRANDO EL MODELO

Si se quiere usar la ORM beego, en todo su contexto, entonces necesitamos obligatoriamente registrar el modelo, para esto lo recomendable es usar un solo archivo models.go, en el que se van a definir las estructuras de la tablas mediante el uso de tipos struct.

Para que el ORM, nos cree las tablas, debemos de crear un objeto tipo struct, con el nombre de la tabla y los nombres de los campos  de la tabla seran los campos de la estructura, algunas de las tablas podran en un caso real necesitar muchos mas campos especificos, pero para el ejemplo, usaremos los datos mas significativos a mi parecer.
```go
	package main

	import (
	"github.com/astaxie/beego/orm"
	)

	type Articulos struct {
	Id          int
	Descripcion string
	}

	type Inventario struct {
	Id        int
	Articulos *Articulos `orm:"rel(fk)"`
	VlrCosto  float64
	Entrada   int
	Salida    int
	}

	func init() {
	orm.RegisterModel(new(Articulos), new(Inventario))
	}
```

el anterior codigo lo vamos a guardar en el archivo models.go.  En este codigo podemos observar que la estructura Inventario tienen  "Articulos *Articulos y se define `orm:"rel(fk)"`, con lo que estamos diciendo que este campo es un campo de relacion a una clave foranea de la tabla articulos.

Al registrar el modelo con orm.RegisterModel(new(Nombre de la tabla)), este va a servir para crear las tablas y es de anotar que los nombres de las tablas seran ceador cambiando su letra inicial de mayuscula a minuscula, lo mismo se hara para los nombres de los campos.

Al registrar el modelo, podemos querer utilizar un prefijo para el nombre de la tabla, para lo que usaremos:

orm.RegisterModelWithPrefix("prefix_", new(Inventario))

el nombre de la tabla se creara como prefix_inventario.
***
Para crear las tablas se puede hacer uso de codigo directamente o de usar comandos por una terminal; se debe de tener presente que de hacerlo por codigo, se deberia desactivar luego de la creacion, ya que de lo contrario cada vez que se corra el codigo, se volvera a crear y se sobre escribiran las tablas o en el caso de que se tenga el parametro `force := false`, no se sobre escribiran pero se hara que el codigo revise cada vez innecesariamente si las tablas existen o no.

Si se quiere por codigo, usaremos:

```go
	func main() {
	o := orm.NewOrm()
	orm.Debug = true
	o.Using("default")

	//Crea las tablas
	name := "default"
	// Borra tablas y las crea nuevamente.
	force := true
	// Print log.
	verbose := true
	// Error.
	err := orm.RunSyncdb(name, force, verbose)
	if err != nil {
		Println(err)
	}
```

Si lo queremos hacer por terminal:

```go
	func main() {
	o := orm.NewOrm()
	orm.Debug = true
	o.Using("default")
	//activa comandos por terminal
	orm.RunCommand()
	}
```
luego en la consola digitamos:

`:~# go build main.go`
`:~# ./relaciones orm syncdb -db="default" -force=1 -v`
***

##OPERACIONES 
Para los siguientes ejemplos puedes optar por introducir los datos a las tablas de forma manual por consola en mysql, o usando phpmyadmin, si lo prefieres graficamente.  Ya mas adelante explico como se pueden introducir los datos mediante codigo.

Podemos usar Insert,Update,Read y Delete, directamente si conocemos la clave primaria:
```go
	o := orm.NewOrm()
	articulos := new(Articulos)
	articulos.Descripcion = "Articulo10"
	o.Insert(articulos)//inserta "Articulo10" en la tabla "articulos" 

	articulos.Id = 10
	o.Delete(articulos)//Borra campo con el "Id=10" de la tabla "articulos"
```
Los siguientes dos trozos de codigo encontrada el registro con el `Id` correspondiente.
```go	
	articulos := new(Articulos)
	articulos.Id = 13
	err = o.Read(articulos)

	if err == orm.ErrNoRows {
		Println("No result found.")
	} else if err == orm.ErrMissPK {
		Println("No primary key found.")
	} else {
		Println(articulos.Descripcion)
	}
```
este codigo hara lo mismo:
```go
	articulos := Articulos{Id: 13}
	err := o.Read(&articulos1)

	if err == orm.ErrNoRows {
		Println("No result found.")
	} else if err == orm.ErrMissPK {
		Println("No primary key found.")
	} else {
		Println(articulos.Descripcion)
	}
```
`Read` siempre usa la primera llave por defecto, si se quiere leer un dato buscando en otra columna, se debe especificar:
```
	articulos := Articulos{Descripcion: "Articulo10"}
	err := o.Read(&articulos, "Descripcion")
	
	if err == orm.ErrNoRows {
		Println("No result found.")
	} else if err == orm.ErrMissPK {
		Println("No primary key found.")
	} else {
		Println(articulos.Id)
	} 
```
`ReadOrCreate` buscara la coincidencia segun el parametro que le hayamos dado, si esta lo muestra de lo contrario, creara el registro
```go
	articulos := Articulos{Descripcion: "Articulo2"}
	if created, id, err := o.ReadOrCreate(&articulos, "Descripcion"); err == nil {
		if created {
			Println("Nuevo objeto insertado. Id:", id)
		} else {
			Println("El objeto esta y es. Id:", id)
		}
	}
```
Podemos con `Insert`, insertar un dato:
```go
	var art Articulos
	art.Descripcion = "articulo15"

	id, err := o.Insert(&art)
	if err == nil {
		Println(id)//el campo id es clave primaria y autoincremental
	}
```
Al inserta multiples datos podemos usar:
```go
	arts := []Articulos{
		{Descripcion: "articulo_za"},
		{Descripcion: "articulo_zb"},
		{Descripcion: "articulo_zc"},
	}
	successNums, err := o.InsertMulti(1, arts)
	Println(successNums, err)
```
si el numero dentro de la funcion `InsertMulti(1, arts)` es mayor a 1, la consulta de insercion de datos los ingresa todos al mismo tiempo, si no los ingresa uno a uno.
***
para actualizar datos:
```go
	articulos := Articulos{Id: 1}
	if o.Read(&articulos) == nil {
	articulos.Descripcion = "Nueva_descripcion"
    	if num, err := o.Update(&articulos); err == nil {
        	Println(num)
    		}
	}

```
Y para borra un dato podemos usar este codigo:
```go
	if num, err := o.Delete(&Articulos{Id: 58}); err == nil {
		Println(num)
	}
```

##CONSULTAS AVANZADAS

-. Buscando registros de una tabla relacionada.
```go
	inventario := &Inventario{}
	o.QueryTable("inventario").Filter("articulos__id", 2).RelatedSel().One(inventario)
	Println(inventario.Articulos.Descripcion)
```	
Podemos encontrar el dato `Descripcion`, del articulo con `Id` 2 que se encuentra en la tabla `inventario`, ya que esta tabla se encuentra relacionada, mediante el parametro `orm:rel(pk)`, que se coloco en el modelo de la estructura de `Inventarios`.

-. Buscando registros en la tabla Inventario, en base al Id de articulos1
```go
	var inven Inventario
	err := o.QueryTable("inventario").Filter("articulos__id", 2).One(&inven)
	if err == nil {
		Println(inven.Entrada)
	}
```
De esta manera podemos usar varios operadores, dentro de la expresiones de la consulta:

Los operadores compatibles:

exacto / iexact igual a
contiene / icontains contiene
GT / GTE mayor que / mayor que o igual a
LT / LTE menos de / menos de o igual a
StartsWith / istartswith comienza con
endswith / iendswith termina con
en
isnull

Los operadores que comienzan con i ignoran caso.

el uso de estos operadores determinan la condicion de la consulta:
```go
	var maps []orm.Params
	num, err := o.QueryTable("inventario").Filter("vlr_costo__gte", 100).Values(&maps)
	if err == nil {
		Printf("Result Nums: %d\n", num)
		for _, m := range maps {
			Println(m["Id"])
		}
	}
```
el resultado sera una lista de los Id de inventario que sean >= a 100

Se pueden usar dos reglas de filtrado, con tener y excluir. Para usar un `and`:
```go
	var maps []orm.Params
	num, err := o.QueryTable("inventario").Filter("vlr_costo__gte", 100).Filter("entrada__gte", 		40).Values	(&maps)
	if err == nil {
		Printf("Result Nums: %d\n", num)
		for _, m := range maps {
			Println(m["Id"])
		}
	}
```
devolvera valores de Id, mayores o iguales a 100, en el Valr_Costo `y` entrada >= 40

tambien se puede escribir:
```go
	var maps []orm.Params

	qs := o.QueryTable("inventario")
	num, err := qs.Filter("vlr_costo__gte", 100).Filter("entrada__gte", 40).Values(&maps)

	if err == nil {
		Printf("Result Nums: %d\n", num)
		for _, m := range maps {
			Println(m["Id"])
		}
	}
```
Para obtener valores en orden ascendente o descendente podemos usar: descendente "-vlr_costo" o ascendente "vlr_costo"
```go	
	var maps []orm.Params
	qs := o.QueryTable("inventario")
	num, err := qs.OrderBy("-vlr_costo").Values(&maps)

	if err == nil {
		Printf("Result Nums: %d\n", num)
		for _, m := range maps {
			Println(m["Id"], m["VlrCosto"])
		}
	}
```
Para hacer consultas relacionales y hacer un join:
```go
	var inventario []*Inventario
	qs := o.QueryTable("inventario")
	num, err := qs.OrderBy("vlr_costo").RelatedSel("articulos").All(&inventario)

	if err == nil {
		Printf("Result Nums: %d\n", num)
		for _, v := range inventario {
			Println(v.Id, v.Articulos.Id, v.Articulos.Descripcion, v.VlrCosto)
		}
	}
```
con el anterior codigo haremos una consulta en la tabla inventario, ordenado ascendentemente  por el "vlr_costo" y relacionaremos la tabla articulos par sacar los datos del articulo.

Con el parametro "All", podemos obtener solo los campos que queremos:
```go
num, err := qs.OrderBy("vlr_costo").RelatedSel("articulos").All(&inventario, "id", "vlr_costo")
```
Se puede observar tambien que en los diferentes codigos puestos anteriormente, van a encontrar una salida; .One(), .All(), .Values(), entre otros, se puede probar segun se necesite la salida y para algunos la variable que toma el resultado es diferente. Para mas informacion sobre las salidas por favor revisar `http://beego.me/docs/mvc/model/query.md`.

SQL Raw para consultar

Si vamos a usar sql raw, no necesitamos registrar el modelo.

Esta consulta ejecutara el update:
```go
	res, err := o.Raw("UPDATE articulos SET descripcion = ? WHERE id=?", "articulo_a",1).Exec()
	if err == nil {
		num, _ := res.RowsAffected()
		Println("mysql row affected nums: ", num)
	}
```
QueryRow: devuelve un dato
```go
	var articulo Articulos
	err := o.Raw("SELECT id, descripcion FROM articulos WHERE id = ?", 1).QueryRow(&articulo)
	if err == nil {
		Println(articulo)
	}
```
QueryRows: devuelve un slice de datos
```go	
	var articulos []Articulos
	num, err := o.Raw("SELECT id, descripcion FROM articulos WHERE id>?", 1).QueryRows(&articulos)
	if err == nil {
		Println(num, err, articulos)
	}
```
SetArgs:podemos usar la misma consulta con diferentes parametros
```go
	o := orm.NewOrm()
	r := o.Raw("UPDATE articulos SET descripcion = ? WHERE id=?")
	res, err := r.SetArgs("art_5", 5).Exec()
	res, err = r.SetArgs("art_6", 6).Exec()
```
Obteniendo datos:	
```go
	o := orm.NewOrm()
	var lists []orm.ParamsList
	num, err := o.Raw("SELECT descripcion FROM articulos WHERE id > ?", 3).ValuesList(&lists)
	if err == nil && num > 0 {
		Println(lists)
	}
```
salida [[art_4] [art_5] [art_6] [articulo_7] [articulo_8]]
```go
	con Println(lists[0][0])
```
salida art_4
```go
	o := orm.NewOrm()
	var list orm.ParamsList
	num, err := o.Raw("SELECT descripcion FROM articulos WHERE id > ?", 3).ValuesFlat(&list)
	if err == nil && num > 0 {
		Println(list)
	}
```
salida [art_4 art_5 art_6 articulo_7 articulo_8]
```go
con Println(list[3])
```
salida articulo_7
```go
	o := orm.NewOrm()
	res := make(orm.Params)
	num, err := o.Raw("SELECT id,descripcion FROM articulos").RowsToMap(&res, "id", "descripcion")
	if err == nil && num > 0 {
		Println(res)
	}
```
map[6:art_6 7:articulo_7 8:articulo_8 1:articulo_a 2:art_2 3:art_3 4:art_4 5:art_5]
```go
con Print(res["5"])
```
salida art_5

Como se puede notar, Los valores de resultados devuelto por la consulta SQL Raw son string.

Puede usar Prepare, para realizar la consulta una vez y luego varias ejecuciones:
```go
	o := orm.NewOrm()
	p, err := o.Raw("UPDATE articulos SET descripcion = ? WHERE id = ?").Prepare()
	res, err := p.Exec("articulo_A", "1")
	res, err = p.Exec("articulo_D", "4")
	
	p.Close()//no olvidarse de cerra el prepare
```
***




	
##NOTAS ADICIONALES:
-. Manual original [BeegoORM](http://beego.me/docs/mvc/model/overview.md)
-. Documentacio [MySQL](http://dev.mysql.com/doc/)
-. sobre el uso de [Paginacion OFFSET]( http://www.cristalab.com/tutoriales/optimizar-consultas-sql-para-paginar-resultados-en-mysql-c92972l/)
-. 

