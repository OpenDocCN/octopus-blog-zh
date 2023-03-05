# 使用 Octopus Deploy 和 Liquibase - Octopus Deploy 部署到 MongoDB

> 原文：<https://octopus.com/blog/octopus-mongodb>

[![Deploying to MongoDB with Octopus Deploy and Liquibase](img/c560830689ec11638b7e6e50f3638057.png)](#)

近年来，使用 NoSQL 作为数据库后端越来越受欢迎。使用率如此之高，以至于亚马逊、微软和谷歌都创建了自己的基于云的产品。在 NoSQL 世界中，一个更容易被认出的名字是 MongoDB。

在本文中，我演示了如何使用 Octopus Deploy 和 Liquibase 自动部署到 MongoDB。

## 液态碱

我之前演示了如何使用 Liquibase 产品将[部署到 Oracle。然而，Liquibase 并不严格在关系数据库空间中运行，他们也有部署到 NoSQL 的解决方案，包括 MongoDB。](https://octopus.com/blog/octopus-oracle-liquibase)

### 变更日志

使用 MongoDB 与其关系计数器部件有很大不同。例如，在关系数据库中创建一个表，在 MongoDB 中创建一个集合。

由于它们差别如此之大，MongoDB 的 Liquibase changelog 也有所不同。

下面包含如何在 MongoDB 中创建集合的示例:

```
<databaseChangeLog

        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xmlns:ext="http://www.liquibase.org/xml/ns/dbchangelog-ext"
        xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-3.6.xsd
        http://www.liquibase.org/xml/ns/dbchangelog-ext http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-ext.xsd">

    <changeSet id="1" author="alex">
            <ext:createCollection collectionName="createCollectionWithValidatorAndOptionsTest">
                <ext:options>
                    {
                    validator: {
                        $jsonSchema: {
                            bsonType: "object",
                            required: ["name", "address"],
                            properties: {
                                name: {
                                    bsonType: "string",
                                    description: "The Name"
                                },
                                address: {
                                    bsonType: "string",
                                    description: "The Address"
                                }
                            }
                        }
                    },
                    validationAction: "warn",
                    validationLevel: "strict"
                    }
                </ext:options>
            </ext:createCollection>
        <ext:createCollection collectionName="createCollectionWithEmptyValidatorTest">
            <ext:options>
            </ext:options>
        </ext:createCollection>
        <ext:createCollection collectionName="createCollectionWithNoValidator"/>
    </changeSet>
</databaseChangeLog> 
```

类似地，向集合中插入数据的语法与 SQL 完全不同。您不是将行插入到表格中，而是将文档插入到集合中。

行和文档之间最大的区别在于，文档不必遵循相同的结构(或关系世界中的模式)，例外情况是集合上有一个验证器，如上所示。

本示例将两个具有不同结构的文档插入到同一集合中:

```
<databaseChangeLog

        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xmlns:ext="http://www.liquibase.org/xml/ns/dbchangelog-ext"
        xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-3.6.xsd
        http://www.liquibase.org/xml/ns/dbchangelog-ext http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-ext.xsd">

    <changeSet id="1" author="alex">
        <ext:insertMany collectionName="insertManyTest1">
            <ext:documents>
                [
                { id: 2 },
                { id: 3,
                  address: { nr: 1, ap: 5}
                }
                ]
            </ext:documents>
        </ext:insertMany>
    </changeSet>
</databaseChangeLog> 
```

使用 Liquibase 的 MongoDB 操作的更多示例可以在这个 [GitHub repo](https://github.com/liquibase/liquibase-mongodb/tree/main/src/test/resources/liquibase/ext) 中找到。

在这篇文章中，我选择了 MongoDB samples 文档中的 AirBnB 示例。我的部署示例将一些数据插入到 Listings 集合中，并创建一个名为 Bookings 的附加集合。清单集合是在我的部署过程中创建的，这将在本文后面解释。

<details><summary>dbchangelog.xml</summary></details>

```
<?xml version="1.1" encoding="UTF-8" standalone="no"?>
<databaseChangeLog

        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xmlns:ext="http://www.liquibase.org/xml/ns/dbchangelog-ext"
        xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-3.6.xsd
        http://www.liquibase.org/xml/ns/dbchangelog-ext http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-ext.xsd">

    <changeSet id="1" author="Shawn.Sesna">
            <ext:createCollection collectionName="Bookings">
            </ext:createCollection>
    </changeSet>
    <changeSet id="2" author="Shawn.Sesna">
        <ext:insertMany collectionName="Listings">
            <ext:documents>
                [
                    {
                    "_id": "10006546",
                    "listing_url": "https://www.airbnb.com/rooms/10006546",
                    "name": "Ribeira Charming Duplex",
                    "summary": "Fantastic duplex apartment with three bedrooms, located in the historic area of Porto, Ribeira (Cube)...",
                    "interaction": "Cot - 10 € / night Dog - € 7,5 / night",
                    "house_rules": "Make the house your home...",
                    "property_type": "House",
                    "room_type": "Entire home/apt",
                    "bed_type": "Real Bed",
                    "minimum_nights": "2",
                    "maximum_nights": "30",
                    "cancellation_policy": "moderate",
                    "last_scraped": {
                        "$date": {
                        "$numberLong": "1550293200000"
                        }
                    },
                    "calendar_last_scraped": {
                        "$date": {
                        "$numberLong": "1550293200000"
                        }
                    },
                    "first_review": {
                        "$date": {
                        "$numberLong": "1451797200000"
                        }
                    },
                    "last_review": {
                        "$date": {
                        "$numberLong": "1547960400000"
                        }
                    },
                    "accommodates": {
                        "$numberInt": "8"
                    },
                    "bedrooms": {
                        "$numberInt": "3"
                    },
                    "beds": {
                        "$numberInt": "5"
                    },
                    "number_of_reviews": {
                        "$numberInt": "51"
                    },
                    "bathrooms": {
                        "$numberDecimal": "1.0"
                    },
                    "amenities": [
                        "TV",
                        "Cable TV",
                        "Wifi",
                        "Kitchen",
                        "Paid parking off premises",
                        "Smoking allowed",
                        "Pets allowed",
                        "Buzzer/wireless intercom",
                        "Heating",
                        "Family/kid friendly",
                        "Washer",
                        "First aid kit",
                        "Fire extinguisher",
                        "Essentials",
                        "Hangers",
                        "Hair dryer",
                        "Iron",
                        "Pack ’n Play/travel crib",
                        "Room-darkening shades",
                        "Hot water",
                        "Bed linens",
                        "Extra pillows and blankets",
                        "Microwave",
                        "Coffee maker",
                        "Refrigerator",
                        "Dishwasher",
                        "Dishes and silverware",
                        "Cooking basics",
                        "Oven",
                        "Stove",
                        "Cleaning before checkout",
                        "Waterfront"
                    ],
                    "price": {
                        "$numberDecimal": "80.00"
                    },
                    "security_deposit": {
                        "$numberDecimal": "200.00"
                    },
                    "cleaning_fee": {
                        "$numberDecimal": "35.00"
                    },
                    "extra_people": {
                        "$numberDecimal": "15.00"
                    },
                    "guests_included": {
                        "$numberDecimal": "6"
                    },
                    "images": {
                        "thumbnail_url": "",
                        "medium_url": "",
                        "picture_url": "https://a0.muscache.com/im/pictures/e83e702f-ef49-40fb-8fa0-6512d7e26e9b.jpg?aki_policy=large",
                        "xl_picture_url": ""
                    },
                    "host": {
                        "host_id": "51399391",
                        "host_url": "https://www.airbnb.com/users/show/51399391",
                        "host_name": "Ana Gonçalo",
                        "host_location": "Porto, Porto District, Portugal",
                        "host_about": "Gostamos de passear, de viajar, de conhecer pessoas e locais novos, gostamos de desporto e animais! Vivemos na cidade mais linda do mundo!!!",
                        "host_response_time": "within an hour",
                        "host_thumbnail_url": "https://a0.muscache.com/im/pictures/fab79f25-2e10-4f0f-9711-663cb69dc7d8.jpg?aki_policy=profile_small",
                        "host_picture_url": "https://a0.muscache.com/im/pictures/fab79f25-2e10-4f0f-9711-663cb69dc7d8.jpg?aki_policy=profile_x_medium",
                        "host_neighbourhood": "",
                        "host_response_rate": {
                        "$numberInt": "100"
                        },
                        "host_is_superhost": false,
                        "host_has_profile_pic": true,
                        "host_identity_verified": true,
                        "host_listings_count": {
                        "$numberInt": "3"
                        },
                        "host_total_listings_count": {
                        "$numberInt": "3"
                        },
                        "host_verifications": [
                        "email",
                        "phone",
                        "reviews",
                        "jumio",
                        "offline_government_id",
                        "government_id"
                        ]
                    },
                    "address": {
                        "street": "Porto, Porto, Portugal",
                        "suburb": "",
                        "government_area": "Cedofeita, Ildefonso, Sé, Miragaia, Nicolau, Vitória",
                        "market": "Porto",
                        "country": "Portugal",
                        "country_code": "PT",
                        "location": {
                        "type": "Point",
                        "coordinates": [
                            {
                            "$numberDouble": "-8.61308"
                            },
                            {
                            "$numberDouble": "41.1413"
                            }
                        ],
                        "is_location_exact": false
                        }
                    },
                    "availability": {
                        "availability_30": {
                        "$numberInt": "28"
                        },
                        "availability_60": {
                        "$numberInt": "47"
                        },
                        "availability_90": {
                        "$numberInt": "74"
                        },
                        "availability_365": {
                        "$numberInt": "239"
                        }
                    },
                    "review_scores": {
                        "review_scores_accuracy": {
                        "$numberInt": "9"
                        },
                        "review_scores_cleanliness": {
                        "$numberInt": "9"
                        },
                        "review_scores_checkin": {
                        "$numberInt": "10"
                        },
                        "review_scores_communication": {
                        "$numberInt": "10"
                        },
                        "review_scores_location": {
                        "$numberInt": "10"
                        },
                        "review_scores_value": {
                        "$numberInt": "9"
                        },
                        "review_scores_rating": {
                        "$numberInt": "89"
                        }
                    },
                    "reviews": [
                        {
                        "_id": "362865132",
                        "date": {
                            "$date": {
                            "$numberLong": "1545886800000"
                            }
                        },
                        "listing_id": "10006546",
                        "reviewer_id": "208880077",
                        "reviewer_name": "Thomas",
                        "comments": "Very helpful hosts. Cooked traditional..."
                        },
                        {
                        "_id": "364728730",
                        "date": {
                            "$date": {
                            "$numberLong": "1546232400000"
                            }
                        },
                        "listing_id": "10006546",
                        "reviewer_id": "91827533",
                        "reviewer_name": "Mr",
                        "comments": "Ana Goncalo were great on communication..."
                        },
                        {  - Server Name: Name or IP address of the MongoDB server
  - Server Port: Port MongoDB is listening on

                        "_id": "403055315",
                        "date": {
                            "$date": {
                            "$numberLong": "1547960400000"
                            }
                        },
                        "listing_id": "10006546",
                        "reviewer_id": "15138940",
                        "reviewer_name": "Milo",
                        "comments": "The house was extremely well located..."
                        }
                    ]
                    }                
                ]
            </ext:documents>
        </ext:insertMany>
    </changeSet>
</databaseChangeLog> 
```

章鱼部署

## 用于部署到 MongoDB 的步骤模板与我用于 Oracle 的模板相同: [Liquibase - Apply changeset](https://library.octopus.com/step-templates/6a276a58-d082-425f-a77a-ff7b3979ce2e/actiontemplate-liquibase-apply-changeset) 。该模板已经更新，将 MongoDB 作为可选的数据库类型。

我的部署项目由以下步骤组成:

创建 MongoDB 数据库

*   服务器名称:MongoDB 服务器的名称或 IP 地址
    *   服务器端口:MongoDB 正在监听的端口
    *   数据库名称:要创建的数据库的名称
    *   初始集合:要创建的集合的名称
    *   用户名:可以创建数据库和集合的帐户的用户名
    *   密码:用户名的密码
    *   创建 MongoDB 用户
*   服务器名称:MongoDB 服务器的名称或 IP 地址
    *   服务器端口:MongoDB 正在监听的端口
    *   管理员用户名:可以创建用户的帐户的用户名
    *   管理员密码:管理员用户名的密码
    *   用户名:要创建的帐户的用户名
    *   密码:用户要创建的密码
    *   添加或更新用户的角色
*   服务器名称:MongoDB 服务器的名称或 IP 地址
    *   服务器端口:MongoDB 正在监听的端口
    *   数据库名称:要授予角色的数据库的名称
    *   管理员用户名:可以创建用户的帐户的用户名
    *   管理员密码:管理员用户名的密码
    *   角色:要分配的以逗号分隔的角色列表
    *   DBA 批准
*   Liquibase -应用变更集
*   Pro 许可证密钥:空
    *   数据库类型:MongoDB
    *   更改日志文件名:dbchangelog.xml
    *   服务器名称:MongoDB 服务器的名称或 IP 地址
    *   服务器端口:MongoDB 正在监听的端口
    *   数据库名称:要应用更新的数据库的名称
    *   用户名:有权更新数据库的用户名
    *   密码:用户帐户的密码
    *   连接查询字符串参数:`?authSource=admin`
    *   数据库驱动程序路径:空
    *   可执行文件路径:空
    *   仅报告？:未选中(NoSQL 数据库不支持)
    *   下载 Liquibase:已检查
    *   Liquibase 版本:空
    *   变更集包:包含 dbchangelog.xml 的包
    *   [![](img/374dd479655ea225d2a4f70410a85e08.png)](#)

MongoDB 要求将`Connection query string parameters`参数设置为提供[认证](https://docs.mongodb.com/manual/reference/connection-string/)的数据库，比如上面显示的`?authSource=admin`。

可以在 Liquibase changelog 中创建数据库、集合和用户，以及为用户分配角色。如果对象还不存在，MongoDB 将简单地动态创建它们。将这些步骤分开是为了让那些不熟悉 MongoDB 工作方式的人更容易理解。

部署

### 部署完成后，我们可以使用 MongoDB Compass 来验证我们的数据库、集合和数据是否已经添加。

[![](img/252e06192afbac17b364b49540fa1ccd.png)](#)

结论

## 在本文中，我展示了使用 Liquibase 和 Octopus Deploy 部署到 MongoDB 是多么容易。

观看网络研讨会

## 我们定期举办网络研讨会。请参见[网络研讨会页面](https://octopus.com/events)，了解过去的网络研讨会和即将举办的网络研讨会的详细信息。

[https://www.youtube.com/embed/1nrxnF4LxGw](https://www.youtube.com/embed/1nrxnF4LxGw)

VIDEO

愉快的部署！

愉快的部署！