---
title: "Encryption in-use: Using CSFLE and Queryable Encryption with Golang"
collection: publications
permalink: /publications/mongodb-atlas-with-terraform
excerpt: 'In this article, we explore how to implement in-use encryption in MongoDB using Golang, focusing on two main approaches: Client-Side Field Level Encryption (CSFLE) and Queryable Encryption. Through a practical script in Golang, we demonstrate how to secure sensitive data, such as employee salary and personal information, ensuring protection throughout the data lifecycle. We cover the differences between CSFLE and Queryable Encryption, highlighting the types of queries each supports and when to use them.'
date: 2024-11-11
# venue: 'MongoDB Developer Center'
# paperurl: 'https://www.mongodb.com/developer/products/atlas/mongodb-atlas-with-terraform'
featuredimage: /images/mongodb-execution-encryption/title.png
featuredimage_alt: "Imagem de cabeçalho"
tags:
  - mongodb
  - golang
  - security
---
# In-Use Encryption: Using CSFLE and Queryable Encryption with Golang

With rising privacy regulations and the need to protect sensitive data, MongoDB provides two advanced encryption solutions that ensure robust security and flexible querying capabilities: **Client-Side Field Level Encryption (CSFLE)** and **Queryable Encryption**. Both options offer end-to-end protection, but they differ significantly in how they encrypt data and the types of queries they support. This article explores the differences between CSFLE and Queryable Encryption, explains how to use them, and demonstrates their implementation in Golang.

## Overview of MongoDB Encryption Types

Before exploring the differences between CSFLE and Queryable Encryption, here are the primary encryption methods offered by MongoDB:

- **Encryption in Transit**: Secures data in motion by encrypting client-server traffic with TLS/SSL. This method is essential for protecting data during network transmission.
- **Encryption at Rest**: Secures data stored on disk, available in MongoDB Enterprise Advanced and Atlas versions. This method ensures data remains protected even in the event of server breaches.
- **Encryption In-Use**: Protects data at all stages of its lifecycle—transmission, storage, and processing. MongoDB offers two approaches for in-use encryption: **Queryable Encryption** and **Client-Side Field Level Encryption (CSFLE)**.

## Differences between CSFLE and Queryable Encryption

### Encryption Method and Inference Security
- **CSFLE**: Uses deterministic encryption for fields that require equality queries, producing identical ciphertexts for identical values. This allows exact match queries but can expose low-cardinality data to inference attacks, where patterns may be identified if values have few variations.
- **Queryable Encryption**: Adopts random encryption, generating unique ciphertexts for identical values, making inference attacks more difficult. This approach supports equality queries and, from MongoDB 8.0 onward, range queries using operators like `$lt`, `$lte`, `$gt`, and `$gte`, making it more robust against low-cardinality data inference.

### Supported Query Types
- **CSFLE**: Supports only exact match (equality) queries on deterministically encrypted fields, suitable for scenarios that don’t require complex query operations. This limitation makes CSFLE ideal for cases where equality is the only required query type.
- **Queryable Encryption**: Offers advanced query support for encrypted data. Besides equality queries, MongoDB 8.0 adds support for range queries with operators like `$lt`, `$lte`, `$gt`, and `$gte`. MongoDB plans to expand this functionality to include prefix, suffix, and substring queries, enhancing flexibility for encrypted data querying.

### When to Use Each Approach
- **CSFLE**: Ideal for scenarios where full control over encryption keys is needed, and equality queries suffice for application requirements. It is particularly useful for highly sensitive data that requires protection throughout client-server communication, offering granular control over the encryption process.
- **Queryable Encryption**: Recommended for applications requiring complex encrypted data queries, such as date ranges or numerical values. It is particularly advantageous for protecting fields with low cardinality, providing greater protection against inference attacks and supporting advanced query operations for sensitive data.

## Implementing CSFLE and Queryable Encryption in Golang

To demonstrate the implementation, we’ll create a simple Golang application to securely manage employee information, storing and querying sensitive data (such as names and salaries) using MongoDB's advanced encryption.

### Prerequisites

The following table shows which MongoDB editions support CSFLE and Queryable Encryption:

| Product Name               | Automatic Encryption Support | Explicit Encryption Support |
|----------------------------|------------------------------|-----------------------------|
| MongoDB Atlas              | Yes                          | Yes                         |
| MongoDB Enterprise Advanced| Yes                          | Yes                         |
| MongoDB Community Edition  | No                           | Yes                         |

Download the Shared Encryption Library from [MongoDB's Shared Library for CSFLE](https://www.mongodb.com/pt-br/docs/v7.0/core/csfle/reference/shared-library/#std-label-csfle-reference-shared-library-download).

Install the Golang module:
```bash
go get libmongocrypt
```

### Default steps

#### Document Structure
Define the `EmployeeDocument` structure to represent each employee's data, including fields such as `Name`, `Position`, `Company`, and `Salary`.

```go
type EmployeeDocument struct {
    Name      string    `bson:"name"`
    Position  string    `bson:"position"`
    Company   string    `bson:"company"`
    Salary    int       `bson:"salary"`
    Currency  string    `bson:"currency"`
    StartDate time.Time `bson:"startDate"`
}
```

#### Load Local Master Key
To encrypt data, we use a master key, which can be loaded from a local file (for demonstration purposes only). The following code checks if the key file exists and creates it if necessary.

To create a new local provider, create a key with this command:

For Shell Unix:

```bash
echo $(head -c 96 /dev/urandom | base64 | tr -d '\n')
```

For PowerShell:

```powershell
$r=[byte[]]::new(64);$g=[System.Security.Cryptography.RandomNumberGenerator]::Create();$g.GetBytes($r);[Convert]::ToBase64String($r)
```

> **Note**: A local key provider is insecure for production. For production environments, use a remote Key Management System (KMS) such as AWS KMS, Azure Key Vault, or Google Cloud KMS, which offer enhanced security and access control.


```go
func loadLocalMasterKey(filename string) string {
	if _, err := os.Stat(filename); os.IsNotExist(err) {
		key := "<YOUR KEY>"
		err = os.WriteFile(filename, []byte(key), 0644)
		if err != nil {
			log.Fatalf("Unable to create the key file: %v", err)
		}
	}
	key, err := os.ReadFile(filename)
	if err != nil {
		log.Fatalf("Unable to read the key file: %v", err)
	}
	return string(key)
}
```

### Implementing Queryable Encryption with Golang

#### Define the variables
Define the variables for the MongoDB connection string, database, and collection names, as well as the key vault namespace and KMS providers.

> Note: Replace `<user>` and `<pass>` with your MongoDB Atlas username and password. In the *GetAutoEncryptionOptions function*, replace the path to the shared encryption library with the correct path on your system.

```go
   uri := "mongodb+srv://<user>:<pass>@cluster0.ag6bk.mongodb.net/?retryWrites=true&w=majority&appName=Cluster0"
   keyVaultDatabaseName := "encryption"
   keyVaultCollectionName := "__keyVault"
   keyVaultNamespace := keyVaultDatabaseName + "." + keyVaultCollectionName
   encryptedDatabaseName := "employee_data"
   encryptedCollectionName := "employee_salary"


   localMasterKeyFile := "local_master_key.txt"
   localMasterKey := loadLocalMasterKey(localMasterKeyFile)


   kmsProviders := map[string]map[string]interface{}{
       "local": {"key": localMasterKey},
   }


   autoEncryptionOptions := GetAutoEncryptionOptions(
       keyVaultNamespace,
       kmsProviders,
       "/Users/samuelmolling/Documents/github/mongodb-lab/queryable-encryption/mongo_crypt_shared_v1-macos-arm64-enterprise-8.0.3/lib/mongo_crypt_v1.dylib",
   )
```

#### Define Encrypted Fields
The `getEncryptedFieldsMap` function specifies which fields will be encrypted and the allowed query types. In this example, `name` supports equality queries, and `salary` supports range queries.

```go
func getEncryptedFieldsMap() bson.M {
	return bson.M{
		"fields": []bson.M{
			{
				"keyId":    nil,
				"path":     "name",
				"bsonType": "string",
				"queries": []bson.M{
					{
						"queryType": "equality",
					},
				},
			},
			{
				"keyId":    nil,
				"path":     "salary",
				"bsonType": "int",
				"queries": []bson.M{
					{
						"queryType": "range",
						"min":       0,
						"max":       1000000,
					},
				},
			},
		},
	}
}
```

#### Configure Automatic Encryption Options
Configure automatic encryption options with the `GetAutoEncryptionOptions` function, specifying the key vault namespace, KMS provider, and shared encryption library path.

```go
func GetAutoEncryptionOptions(keyVaultNamespace string, kmsProviders map[string]map[string]interface{}, cryptSharedLibPath string) *options.AutoEncryptionOptions {
	extraOptions := map[string]interface{}{
		"cryptSharedLibPath": cryptSharedLibPath,
	}
	return options.AutoEncryption().
		SetKeyVaultNamespace(keyVaultNamespace).
		SetKmsProviders(kmsProviders).
		SetExtraOptions(extraOptions)
}
```

#### Create MongoDB Client with Automatic Encryption
Create a MongoDB client with automatic encryption enabled.

```go
clientEncryptionOpts := options.ClientEncryption().
    SetKmsProviders(kmsProviders).
    SetKeyVaultNamespace(keyVaultNamespace)
clientEncryption, err := mongo.NewClientEncryption(encryptedClient, clientEncryptionOpts)
if err != nil {
    log.Fatalf("Unable to create ClientEncryption instance: %v", err)
}
defer clientEncryption.Close(context.Background())
```

#### Set Up Key Vault and Encrypted Collection
Configure the key vault and create the encrypted collection with `CreateEncryptedCollection`.

```go
func (ce *ClientEncryption) CreateEncryptedCollection(ctx context.Context,
	db *Database, coll string, createOpts *options.CreateCollectionOptions,
	kmsProvider string, masterKey interface{}) (*Collection, bson.M, error) {
	if createOpts == nil {
		return nil, nil, errors.New("nil CreateCollectionOptions")
	}
	ef := createOpts.EncryptedFields
	if ef == nil {
		return nil, nil, errors.New("no EncryptedFields defined for the collection")
	}

	efBSON, err := marshal(ef, db.bsonOpts, db.registry)
	if err != nil {
		return nil, nil, err
	}
	r := bsonrw.NewBSONDocumentReader(efBSON)
	dec, err := bson.NewDecoder(r)
	if err != nil {
		return nil, nil, err
	}
	var m bson.M
	err = dec.Decode(&m)
	if err != nil {
		return nil, nil, err
	}

	if v, ok := m["fields"]; ok {
		if fields, ok := v.(bson.A); ok {
			for _, field := range fields {
				if f, ok := field.(bson.M); !ok {
					continue
				} else if v, ok := f["keyId"]; ok && v == nil {
					dkOpts := options.DataKey()
					if masterKey != nil {
						dkOpts.SetMasterKey(masterKey)
					}
					keyid, err := ce.CreateDataKey(ctx, kmsProvider, dkOpts)
					if err != nil {
						createOpts.EncryptedFields = m
						return nil, m, err
					}
					f["keyId"] = keyid
				}
			}
			createOpts.EncryptedFields = m
		}
	}
	err = db.CreateCollection(ctx, coll, createOpts)
	if err != nil {
		return nil, m, err
	}
	return db.Collection(coll), m, nil
}

createCollectionOptions := options.CreateCollection().SetEncryptedFields(encryptedFieldsMap)
_, _, err = clientEncryption.CreateEncryptedCollection(
    context.TODO(),
    encryptedClient.Database(encryptedDatabaseName),
    encryptedCollectionName,
    createCollectionOptions,
    "local",
    map[string]string{},
)
```

![](/images/mongodb-execution-encryption/coll-query-encryption.png)
![](/images/mongodb-execution-encryption/index-query-encryption.png)


#### Insert Encrypted Documents
Insert example documents into the encrypted collection, with sensitive information protected by encryption.

```go
employees := []EmployeeDocument{
  {"Alice Johnson", "Software Engineer", "MongoDB", 100000, "USD", time.Date(2019, time.March, 5, 0, 0, 0, 0, time.UTC)},
  {"Bob Smith", "Product Manager", "MongoDB", 150000, "USD", time.Date(2018, time.June, 15, 0, 0, 0, 0, time.UTC)},
  {"Charlie Brown", "Data Analyst", "MongoDB", 200000, "USD", time.Date(2020, time.April, 20, 0, 0, 0, 0, time.UTC)},
  {"Diana Prince", "HR Specialist", "MongoDB", 250000, "USD", time.Date(2021, time.December, 3, 0, 0, 0, 0, time.UTC)},
  {"Evan Peters", "Marketing Coordinator", "MongoDB", 80000, "USD", time.Date(2022, time.October, 7, 0, 0, 0, 0, time.UTC)},
}

coll := encryptedClient.Database(encryptedDatabaseName).Collection(encryptedCollectionName)
for _, employee := range employees {
  _, err = coll.InsertOne(context.TODO(), employee)
  if err != nil {
    log.Fatalf("Unable to insert the employee document: %s", err)
  }
  fmt.Printf("Inserted document for %s\n", employee.Name)
}
```

![](/images/mongodb-execution-encryption/result-ui-atlas-query-encryption.png)

Without the key we can't see the data. It worked.


#### Perform Encrypted Queries
Finally, perform queries on the encrypted collection. The `searchByName` function searches documents by name, while `searchBySalaryRange` uses a salary range filter.

```go
coll = encryptedClient.Database("employee_data").Collection("employee_salary")

searchByName(coll, "Alice Johnson")
searchBySalaryRange(coll, 150000, 200000)
```

![](/images/mongodb-execution-encryption/result-query-encryption.png)

As we can see, in version 8, I can also do this with data ranges.

You can check the entire code in the [GitHub repository](https://github.com/SamuelMolling/mongodb-lab/blob/main/queryable-encryption/main.go) and more informations in the [MongoDB documentation](https://www.mongodb.com/docs/manual/core/queryable-encryption/).

### Implementing CSFLE with Golang

#### Define the variables
Define the variables for the MongoDB connection string, database, and collection names, as well as the key vault namespace and KMS providers.

> Note: Replace `<user>` and `<pass>` with your MongoDB Atlas username and password. In the *GetAutoEncryptionOptions function*, replace the path to the shared encryption library with the correct path on your system.

```go
func setupKMSProviders(localMasterKey string) map[string]map[string]interface{} {
	return map[string]map[string]interface{}{
		"local": {"key": localMasterKey},
	}
}

uri := "mongodb+srv://<user>:<pass>@demo1.f7x641l.mongodb.net/?retryWrites=true&w=majority&appName=demo1"
localMasterKey := "<YOUR KEY>"
kmsProviders := setupKMSProviders(localMasterKey)
keyVaultNamespace := "encryption.__keyVault"
```

#### Ensuring Key Vault Index

We ensure the keyAltNames index exists in the Key Vault, allowing alternate keys to be unique and filtered.

```go
func ensureKeyVaultIndex(keyVaultColl *mongo.Collection) {
	indexName := "keyAltNames_1"
	cursor, err := keyVaultColl.Indexes().List(context.TODO())
	if err != nil {
		log.Fatalf("Error listing indexes: %v", err)
	}
	defer cursor.Close(context.TODO())

	exists := false
	for cursor.Next(context.TODO()) {
		var index bson.M
		if err := cursor.Decode(&index); err != nil {
			log.Fatalf("Error decoding index: %v", err)
		}
		if index["name"] == indexName {
			exists = true
			break
		}
	}

	if !exists {
		keyVaultIndex := mongo.IndexModel{
			Keys: bson.D{{Key: "keyAltNames", Value: 1}},
			Options: options.Index().
				SetUnique(true).
				SetPartialFilterExpression(bson.D{
					{Key: "keyAltNames", Value: bson.D{
						{Key: "$exists", Value: true},
					}},
				}),
		}
		_, err = keyVaultColl.Indexes().CreateOne(context.TODO(), keyVaultIndex)
		if err != nil {
			log.Fatalf("Error creating index in key vault: %v", err)
		}
		fmt.Println("Index created in key vault.")
	} else {
		fmt.Println("Index already exists in key vault.")
	}
}
```

#### Generating and Reusing Data Key
The *ensureDataKey* function checks if a specific data key already exists. If it doesn’t, it creates a new one.


```go
func ensureDataKey(clientEncryption *mongo.ClientEncryption, keyVaultColl *mongo.Collection, keyAltName string) (primitive.Binary, error) {
	var existingKey bson.M
	err := keyVaultColl.FindOne(context.TODO(), bson.M{"keyAltNames": keyAltName}).Decode(&existingKey)
	if err == nil {
		fmt.Println("Data key already exists. Reusing existing key.")
		return existingKey["_id"].(primitive.Binary), nil
	} else if err != mongo.ErrNoDocuments {
		return primitive.Binary{}, err
	}

	fmt.Println("Creating new data key.")
	dataKeyOpts := options.DataKey().SetKeyAltNames([]string{keyAltName})
	dataKeyID, err := clientEncryption.CreateDataKey(context.TODO(), "local", dataKeyOpts)
	if err != nil {
		return primitive.Binary{}, err
	}
	return dataKeyID, nil
}
```

#### Encrypting the Salary
The encryptSalary function converts the salary to cents, prepares the value for encryption, and encrypts it using the configured MongoDB encryption algorithm.

```go
func encryptSalary(clientEncryption *mongo.ClientEncryption, dataKeyID primitive.Binary, salary float64) primitive.Binary {
	salaryInCents := int64(salary * 100)
	rawValueType, rawValueData, err := bson.MarshalValue(salaryInCents)
	if err != nil {
		log.Fatalf("Error preparing value for encryption: %v", err)
	}
	rawValue := bson.RawValue{Type: rawValueType, Value: rawValueData}

	encryptionOpts := options.Encrypt().
		SetAlgorithm("AEAD_AES_256_CBC_HMAC_SHA_512-Deterministic").
		SetKeyID(dataKeyID)

	encryptedData, err := clientEncryption.Encrypt(context.TODO(), rawValue, encryptionOpts)
	if err != nil {
		log.Fatalf("Error encrypting salary: %v", err)
	}

	return primitive.Binary{Subtype: encryptedData.Subtype, Data: encryptedData.Data}
}
```

#### Inserting Encrypted Documents
After encrypting the salary, we insert each employee document into the collection. The document includes fields such as Name, Position, Company, Currency, Start Date, and the encrypted Salary.


```go
func insertEmployeeDoc(coll *mongo.Collection, name, position, company string, salaryEncrypted primitive.Binary, currency string, startDate time.Time) {
	employeeDoc := bson.D{
		{Key: "name", Value: name},
		{Key: "position", Value: position},
		{Key: "company", Value: company},
		{Key: "salary", Value: salaryEncrypted},
		{Key: "currency", Value: currency},
		{Key: "startDate", Value: startDate},
	}

	_, err := coll.InsertOne(context.TODO(), employeeDoc)
	if err != nil {
		log.Fatalf("Error inserting employee document: %v", err)
	}
}

employees := []struct {
		Name      string
		Position  string
		Company   string
		Salary    float64
		Currency  string
		StartDate time.Time
	}{
		{"Alice Johnson", "Software Engineer", "MongoDB", 50000, "USD", time.Date(2007, time.February, 3, 0, 0, 0, 0, time.UTC)},
		{"Bob Smith", "Product Manager", "MongoDB", 70000, "USD", time.Date(2009, time.March, 14, 0, 0, 0, 0, time.UTC)},
		{"Charlie Brown", "Data Analyst", "MongoDB", 90000, "USD", time.Date(2011, time.June, 21, 0, 0, 0, 0, time.UTC)},
		{"Diana Prince", "Project Manager", "MongoDB", 110000, "USD", time.Date(2012, time.July, 11, 0, 0, 0, 0, time.UTC)},
		{"Edward Stark", "DevOps Engineer", "MongoDB", 130000, "USD", time.Date(2013, time.August, 9, 0, 0, 0, 0, time.UTC)},
		{"Fiona Gallagher", "HR Specialist", "MongoDB", 150000, "USD", time.Date(2014, time.September, 12, 0, 0, 0, 0, time.UTC)},
		{"George Orwell", "Security Analyst", "MongoDB", 170000, "USD", time.Date(2015, time.October, 22, 0, 0, 0, 0, time.UTC)},
		{"Hannah Montana", "Marketing Coordinator", "MongoDB", 190000, "USD", time.Date(2016, time.November, 19, 0, 0, 0, 0, time.UTC)},
		{"Isaac Newton", "Chief Scientist", "MongoDB", 210000, "USD", time.Date(2016, time.December, 5, 0, 0, 0, 0, time.UTC)},
		{"Julia Roberts", "Finance Manager", "MongoDB", 250000, "USD", time.Date(2008, time.January, 28, 0, 0, 0, 0, time.UTC)},
	}

for _, emp := range employees {
  encryptedSalary := encryptSalary(clientEncryption, dataKeyID, emp.Salary)
  insertEmployeeDoc(coll, emp.Name, emp.Position, emp.Company, encryptedSalary, emp.Currency, emp.StartDate)
}
```

![](/images/mongodb-execution-encryption/result-ui-csfle.png)

Without the key we can't see the data. It worked.

#### Querying and Decrypting Documents
To query and view the encrypted salary field, we use findAllAndDecryptSalaries, which retrieves all documents, decrypts the salary, and displays the data.

```go
func findAllAndDecryptSalaries(coll *mongo.Collection, clientEncryption *mongo.ClientEncryption) {
	cursor, err := coll.Find(context.TODO(), bson.D{})
	if err != nil {
		log.Fatalf("Error finding documents: %v", err)
	}
	defer cursor.Close(context.TODO())

	for cursor.Next(context.TODO()) {
		var foundDoc bson.M
		if err := cursor.Decode(&foundDoc); err != nil {
			log.Fatalf("Error decoding document: %v", err)
		}

		decrypted, err := clientEncryption.Decrypt(context.TODO(), foundDoc["salary"].(primitive.Binary))
		if err != nil {
			log.Fatalf("Error decrypting salary: %v", err)
		}

		var decryptedSalary int64
		if err := decrypted.Unmarshal(&decryptedSalary); err != nil {
			log.Fatalf("Error unmarshaling decrypted salary: %v", err)
		}

		salaryInDollars := float64(decryptedSalary) / 100.0

		startDate := foundDoc["startDate"].(primitive.DateTime).Time().Format("2006-01-02")

		fmt.Printf("Employee: %s\n", foundDoc["name"])
		fmt.Printf("Position: %s\n", foundDoc["position"])
		fmt.Printf("Company: %s\n", foundDoc["company"])
		fmt.Printf("Start Date: %s\n", startDate)
		fmt.Printf("Currency: %s\n", foundDoc["currency"])
		fmt.Printf("Decrypted Salary: %.2f USD\n", salaryInDollars)
		fmt.Println("---------------------------------------------------")
	}

	if err := cursor.Err(); err != nil {
		log.Fatalf("Cursor error: %v", err)
	}
}
```

For comparison, the findAllWithoutDecryption function retrieves the same documents but without decrypting the salaries.

```go
func findAllWithoutDecryption(coll *mongo.Collection) {
	cursor, err := coll.Find(context.TODO(), bson.D{})
	if err != nil {
		log.Fatalf("Error finding documents: %v", err)
	}
	defer cursor.Close(context.TODO())

	for cursor.Next(context.TODO()) {
		var foundDoc bson.M
		if err := cursor.Decode(&foundDoc); err != nil {
			log.Fatalf("Error decoding document: %v", err)
		}

		startDate := foundDoc["startDate"].(primitive.DateTime).Time().Format("2006-01-02")

		fmt.Printf("Employee: %s\n", foundDoc["name"])
		fmt.Printf("Position: %s\n", foundDoc["position"])
		fmt.Printf("Company: %s\n", foundDoc["company"])
		fmt.Printf("Start Date: %s\n", startDate)
		fmt.Printf("Currency: %s\n", foundDoc["currency"])
		fmt.Printf("Encrypted Salary: %v\n", foundDoc["salary"])
		fmt.Println("---------------------------------------------------")
	}

	if err := cursor.Err(); err != nil {
		log.Fatalf("Cursor error: %v", err)
	}
}
```
The result is: 

![](/images/mongodb-execution-encryption/result-csfle.png)

Let's zoom in and see the difference between two objects, one encrypted and the other not.

![](/images/mongodb-execution-encryption/result-csfle-1.png)

As we can see in this image, the real value of the object that is used without encryption comes in a format that is unreadable to us without the encryption key.

You can check the entire code in the [GitHub repository](https://github.com/SamuelMolling/mongodb-lab/blob/main/csfle/main.go) and more informations in the [MongoDB documentation](https://www.mongodb.com/docs/manual/core/csfle/).

## Conclusion

CSFLE and Queryable Encryption are advanced encryption solutions in MongoDB, providing distinct methods for protecting sensitive data and enabling secure queries. CSFLE is ideal for cases where client-side control and equality queries are sufficient, while Queryable Encryption excels in scenarios requiring advanced queries like ranges and complex comparisons. With MongoDB 8.0's range query support, Queryable Encryption becomes even more powerful and flexible for securing sensitive data.

Using Golang, you can easily configure and use these encryption solutions to enhance MongoDB application security, meeting strict data compliance and security requirements.

For more MongoDB resources and tools, visit the MongoDB Developer Center to explore additional articles.