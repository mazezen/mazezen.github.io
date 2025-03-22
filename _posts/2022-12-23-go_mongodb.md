---
layout: post
title: "Go操作MongoDB"
date:    2023-12-23
tags: [Go]
comments: true
author: mazezen
---

​	**驱动安装包**

```go
go get go.mongodb.org/mongo-driver/mongo
```

### 定义uri、database、collection

```go
var (
	uri        = "mongodb://127.0.0.1:27017/?maxPoolSize=20&w=majority"
	mon        *mongo.Client
	dataBase   = "echo-scaffolding" // 数据库
	collection = "restaurants"
)
```



###  1.创建一个mongodb客户端

```go
func NewMongoDB() *mongo.Client {
	clientOptions := options.Client().ApplyURI(uri)

	client, err := mongo.Connect(context.TODO(), clientOptions)
	if err != nil {
		panic(err)
	}

	if err = client.Ping(context.TODO(), nil); err != nil {
		panic(err)
	}
	fmt.Println("successfully connected and pinged.")
	return client
}
```



### 2. 写入

```go
func main() {
	mon = NewMongoDB()
	coll := mon.Database(dataBase).Collection(collection)
	newRestaurant := Restaurant{Name: "8282", Cuisine: "korean"}
	result, err := coll.InsertOne(context.TODO(), newRestaurant)
	if err != nil {
		fmt.Println(err)
		os.Exit(1)
	}
	fmt.Printf("Document inserted with ID： %s\n", result.InsertedID)
}
```

```go
func main() {
	mon = NewMongoDB()
	coll := mon.Database(dataBase).Collection(collection)
	newRestaurants := []interface{}{
		Restaurant{Name: "Rule of thirds", Cuisine: "Japanese"},
		Restaurant{Name: "Madame Vo", Cuisine: "Vietnamese"},
	}
	result, err := coll.InsertMany(context.TODO(), newRestaurants)
	if err != nil {
		fmt.Println(err)
		os.Exit(1)
	}
	fmt.Printf("%d documents inseted with IDs:\n", len(result.InsertedIDs))
	for _, id := range result.InsertedIDs {
		fmt.Printf("\t%s\n", id)
	}
}
```

### 3.  查找

```go
func main() {
	mon = NewMongoDB()
	coll := mon.Database(dataBase).Collection(collection)
	filter := bson.D{{"name", "Rule of thirds"}}
	var result Restaurant
	err := coll.FindOne(context.TODO(), filter).Decode(&result)
	if err != nil {
		if err == mongo.ErrNoDocuments {
			fmt.Println("not match any documents")
		}
		fmt.Println(err)
		os.Exit(1)
	}
	output, err := json.MarshalIndent(result, "", "    ")
	if err != nil {
		fmt.Println(err)
		os.Exit(1)
	}
	fmt.Printf("%s\n", output)
}
```

```go
func main() {
	mon = NewMongoDB()
	coll := mon.Database(dataBase).Collection(collection)
	filter := bson.D{{"cuisine", "Japanese"}}
	cursor, err := coll.Find(context.TODO(), filter)
	if err != nil {
		fmt.Println(err)
		os.Exit(1)
	}

	var results []Restaurant
	if err = cursor.All(context.TODO(), &results); err != nil {
		fmt.Println(err)
		os.Exit(1)
	}

	for _, result := range results {
		cursor.Decode(&result)
		output, err := json.MarshalIndent(result, "", "    ")
		if err != nil {
      fmt.Println(err)
      os.Exit(1)
		}
		fmt.Printf("%s\n", output)
	}
}
```

### 4.  更新

```go
func main() {
	mon = NewMongoDB()
	coll := mon.Database(dataBase).Collection(collection)
	id, _ := primitive.ObjectIDFromHex("63a5696412b571d2b15db3e4")
	filter := bson.D{{"_id", id}}
	update := bson.D{{"$set", bson.D{{"avg_rating", 4.4}}}}

	result, err := coll.UpdateOne(context.TODO(), filter, update)
	if err != nil {
      fmt.Println(err)
      os.Exit(1)
	}
	fmt.Printf("Documents updated: %v\n", result.ModifiedCount)
}
```

### 5. 统计

```go
func main() {
	mon = NewMongoDB()
	coll := mon.Database(dataBase).Collection(collection)
	filter := bson.D{{"name", "8282"}}
	estCount, err := coll.EstimatedDocumentCount(context.TODO())
	if err != nil {
      fmt.Println(err)
      os.Exit(1)
	}
	count, err := coll.CountDocuments(context.TODO(), filter)
	if err != nil {
      fmt.Println(err)
      os.Exit(1)
	}
	fmt.Printf("Estimated number of documents in the restaurants collection: %d\n", estCount)
	fmt.Printf("Number of name Japanese 8282: %d\n", count)
}
```

### 6. 删除

```go
func main() {
	mon = NewMongoDB()
	coll := mon.Database(dataBase).Collection(collection)
	filter := bson.D{{"name", "8282"}}
	result, err := coll.DeleteOne(context.TODO(), filter)
	if err != nil {
      fmt.Println(err)
      os.Exit(1)
	}
	fmt.Printf("Documents deleted: %d\n", result.DeletedCount)
}
```

