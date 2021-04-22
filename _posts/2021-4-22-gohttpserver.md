---
title: 一个简单的Go语言http server
categories:
- Go
---


#### 1. HelloWorld

首先我们建立一个迷你服务器，在`localhost:8000`上显示Hello, World.

```Go
import(
	"fmt"
	"log"
	"net/http"
	"github.com/gorilla/mux"
)

func main() {
	router := mux.NewRouter()
	router.HandleFunc("/", index)
	
	log.Fatal(http.ListenAndServe("localhost:8000", router))
}

func index(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hello, World!")
}
```

好像没有太多要解释的，因为工作都让库函数做完了。这里用了`github.com/gorilla/mux`这个包而不是书上的http自带，因为这个包可以更方便的管理http method，不然就要类似`switch r.Method`这样，不是很美观。现在`go run server.go &`，打开浏览器就可以在对应地址看到hw了。

#### 2. 几个简单的接口

接下来加入一些Restful接口。具体如下：

```Go
func main() {
	router := mux.NewRouter()
	router.HandleFunc("/", index)
	router.HandleFunc("/accounts", createAccount).Methods("POST")
	router.HandleFunc("/accounts/{id}", deleteAccount).Methods("DELETE")
	router.HandleFunc("/accounts", updateAccount).Methods("PUT")
	router.HandleFunc("/accounts", getAccount).Methods("GET")
	router.HandleFunc("/accounts/{id}", getAccountByID).Methods("GET")
	
	log.Fatal(http.ListenAndServe("localhost:8000", router))
}
```

#### 3.连接数据库

接下来我们需要引入mysql，首先导入包：
```Go
import(
	"database/sql"
    _ "github.com/go-sql-driver/mysql"
    "encoding/json"
    "strconv"
)
```
定义数据结构来对应数据库：

```Go
type Account struct {
	Id int
	Name string
	Amount int
}
```

写了如下的辅助函数来连接数据库：

```Go
func connectDB() *sql.DB {
	db, err := sql.Open("mysql", "username:password@tcp(127.0.0.1:3306)/dbname")
	if err != nil {
		fmt.Println(err)
		panic(err.Error())
	}
	fmt.Println("Successfully connected to database.")
	return db
}
```

其中`username:password@tcp(127.0.0.1:3306)/dbname`表示了用户名、密码、地址跟数据库名字的格式。

#### 4.增删改查
新增一个账号，这里在RequestBody里填写一个json表单，然后用`json.Decoder.Decode`读取到Go的数据结构里。用`db.Prepare`生成mysql statement，然后用`stmt.Exec`执行。需要研究的地方是怎么设置id自增而不是手动填写。
```Go
func createAccount(w http.ResponseWriter, r *http.Request) {
	db := connectDB()

	var account Account
	decoder := json.NewDecoder(r.Body)
	err := decoder.Decode(&account)

	stmt, err := db.Prepare("INSERT into account SET Id=?, Name=?, Amount=?")
	if err != nil {
		fmt.Println(err)
	}

	_, err2 := stmt.Exec(account.Id, account.Name, account.Amount)
	if err2 != nil {
		fmt.Println(err2)
	}
}
```

删除账号，这里通过url里的id来进行删除，所以用了`mux.Vars`跟`strconv`来获取id。
```Go
func deleteAccount(w http.ResponseWriter, r *http.Request) {
	db := connectDB()
	params := mux.Vars(r)
	id, _ := strconv.Atoi(params["id"])

	stmt, err := db.Prepare("DELETE FROM account WHERE id=?")
	if err != nil{
		fmt.Println(err)
	}

	result, err := stmt.Exec(id)
	if err != nil{
		fmt.Println(err)
	}
	json, _ := json.Marshal(result)
	w.Write(json)
}
```

更新账号信息，同样是通过RequestBody。这里通过`RowsAffected`判断有没有处理到，讲道理并不是最好的解法，因为并不一定是没有这个id才会发生这件事，应该先查找有没有对应的id，只不过会引入新的开销。
```Go
func updateAccount(w http.ResponseWriter, r *http.Request) {
	db := connectDB()
	var account Account
	decoder := json.NewDecoder(r.Body)
	err := decoder.Decode(&account)

	stmt, err := db.Prepare("UPDATE account SET Name=?, Amount=? WHERE id=?")
	if err != nil{
		fmt.Println(err)
	}

	result, err := stmt.Exec(account.Name, account.Amount, account.Id)
	if err != nil{
		fmt.Println(err)
	}

	cnt, _ := result.RowsAffected();
	if(cnt == 0) {
		w.Write([]byte("Could not find account with id: " + strconv.Itoa(account.Id)))
	} else {
		fmt.Println(result)
	}
}
```

最后是查找，首先是展示全部的账号信息，会返回`sql.Rows`然后进行遍历，把结果存到Account的slice里，用`json.Marshal`输出成可以给`w.Write`的`[]byte`类型。
```Go
func getAccount(w http.ResponseWriter, r *http.Request) {
	db := connectDB()

	rows, err := db.Query("SELECT * FROM account")
	if err != nil{
		fmt.Println(err)
	}

	results := []Account{}
	account := Account{}

	for rows.Next() {
        e := rows.Scan(&account.Id, &account.Name, &account.Amount)
        if e != nil {
            fmt.Println(err)
        }
        results = append(results, account)
    }

    rows.Close()
	json, _ := json.Marshal(results)
	w.Write(json)
}
```

还写了一个通过id查找，原理类似。显然，可以模块化一下输出的部分。
```Go
func getAccountByID(w http.ResponseWriter, r *http.Request) {
	db := connectDB()
	params := mux.Vars(r)
	id, _ := strconv.Atoi(params["id"])

	rows, err := db.Query("SELECT * FROM account WHERE Id=?", id)
	if err != nil {
		fmt.Println(err)
	}

	results := []Account{}
	account := Account{}

	if rows.Next() {
        e := rows.Scan(&account.Id, &account.Name, &account.Amount)
        if e != nil {
            fmt.Println(err)
        }
        results = append(results, account)
    } else {
    	w.Write([]byte("Could not find account with id: " + params["id"]))
    	return
    }

    rows.Close()
	json, _ := json.Marshal(results)
	w.Write(json)
}
```

#### 5.测试
用postman简单测试了一下各个接口，但是~~懒得~~忘记截图了！

那么祝您身体健康

