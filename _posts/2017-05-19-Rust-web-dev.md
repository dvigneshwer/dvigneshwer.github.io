---
layout: post
title: 'Web Development in Rust'
date: 2017-05-19
permalink: /posts/2017/05/Rust-web-dev/
tags:
  - Rust
  - JWT
  - MongoDB
  - Web Development
  - REST API
---

This blog is gonna briefly cover all the aspect about this web teaching kit which I have curated for the RainOfRust campaign.

Below are the key takeways and learning from this activity.

Key learnings:

* Learn to build web services in Rust
* Integrate db services

Key takeaways:

* Code samples and instructions for building the service

## Instructions

Follow the stepwise instruction below to setup the whole project:

* Create the project

~~~~
cargo new --bin rust-users
~~~~

* Mention the dependencies

~~~~
nano Cargo.toml

[package]

name = "rust-users"
version = "0.0.1"
authors = [ "Firstname Lastname <dvigneshwer@gmail.com>" ]

[dependencies]
nickel = "*"
mongodb = "*"
bson = "*"
rustc-serialize = "*"

~~~~

* Copy and paste the following code in main.rs:

~~~~
#[macro_use]
extern crate nickel;
extern crate rustc_serialize;

#[macro_use(bson, doc)]
extern crate bson;
extern crate mongodb;

// Nickel
use nickel::{Nickel, JsonBody, HttpRouter, MediaType};
use nickel::status::StatusCode::{self};

// MongoDB
use mongodb::{Client, ThreadedClient};
use mongodb::db::ThreadedDatabase;
use mongodb::error::Result as MongoResult;

// bson
use bson::{Bson, Document};
use bson::oid::ObjectId;

// rustc_serialize
use rustc_serialize::json::{Json, ToJson};

#[derive(RustcDecodable, RustcEncodable)]
struct User {
    firstname: String,
    lastname: String,
    email: String
}

fn main() {

    let mut server = Nickel::new();
    let mut router = Nickel::router();

    router.get("/users", middleware! { |request, response|

	    // Connect to the database
	    let client = Client::connect("localhost", 27017)
	      .ok().expect("Error establishing connection.");

	    // The users collection
	    let coll = client.db("rust-users").collection("users");

	    // Create cursor that finds all documents
	    let mut cursor = coll.find(None, None).unwrap();

	    // Opening for the JSON string to be returned
	    let mut data_result = "{\"data\":[".to_owned();

	    for (i, result) in cursor.enumerate() {
	        
	    	if let Ok(item) = result {
	    	        if let Some(&Bson::String(ref firstname)) = item.get("firstname") {
	    	           
	    	            let string_data = if i == 0 {
    	                    format!("{},", firstname)
    	                } else {
    	                    format!("{},", firstname)
    	                };
    	                data_result.push_str(&string_data);
	    	        }

	    	        
	    	    }
	    }

	    // Close the JSON string
	    data_result.push_str("]}");

	    // Send back the result
	    format!("{}", data_result)

	});

	router.post("/users/new", middleware! { |request, response|

	    // Accept a JSON string that corresponds to the User struct
	    let user = request.json_as::<User>().unwrap();

	    let firstname = user.firstname.to_string();
	    let lastname = user.lastname.to_string();
	    let email = user.email.to_string();

	    // Connect to the database
	    let client = Client::connect("localhost", 27017)
	        .ok().expect("Error establishing connection.");

	    // The users collection
	    let coll = client.db("rust-users").collection("users");

	    // Insert one user
	    match coll.insert_one(doc! {
	        "firstname" => firstname,
	        "lastname" => lastname,
	        "email" => email
	    }, None) {
	        Ok(_) => (StatusCode::Ok, "Item saved!"),
	        Err(e) => return response.send(format!("{}", e))
	    }

	});
	
	router.delete("/users/:id", middleware! { |request, response|

	    let client = Client::connect("localhost", 27017)
	        .ok().expect("Failed to initialize standalone client.");

	    // The users collection
	    let coll = client.db("rust-users").collection("users");

	    // Get the objectId from the request params
	    let object_id = request.param("id").unwrap();

	    // Match the user id to an bson ObjectId
	    let id = match ObjectId::with_string(object_id) {
	        Ok(oid) => oid,
	        Err(e) => return response.send(format!("{}", e))
	    };

	    match coll.delete_one(doc! {"_id" => id}, None) {
	        Ok(_) => (StatusCode::Ok, "Item deleted!"),
	        Err(e) => return response.send(format!("{}", e))
	    }

	});

    server.utilize(router);

    server.listen("127.0.0.1:9000");
}
~~~~

* Setup mongodb:

~~~~
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 7F0CEB10

echo "deb http://repo.mongodb.org/apt/ubuntu "$(lsb_release -sc)"/mongodb-org/3.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-3.0.list

sudo apt-get update

sudo apt-get install -y mongodb-org

service mongod status
~~~~~

* Check if its working fine:

~~~~
$ mongo
> use rust-users
> db.users.find( { "firstname": "VIKI" } ).pretty()
~~~~

* Run the program:

~~~~
cargo run
~~~~

We should get theses set of [outputs](https://github.com/MozillaIndia/RustIndia/blob/master/RainOfRust/Readme.md#output--2) on hitting these endpoints.

Ref :

* [Source code](https://github.com/MozillaIndia/RustIndia/tree/master/RainOfRust/rust-users)
* [Build an API in Rust with JWT Authentication using Nickel.rs](https://auth0.com/blog/build-an-api-in-rust-with-jwt-authentication-using-nickelrs/)
* [Mongo Rust driver](https://github.com/mongodb-labs/mongo-rust-driver-prototype)
* [Nickle framework](http://nickel.rs/)
* [Web development source for Rust](http://www.arewewebyet.org/)
