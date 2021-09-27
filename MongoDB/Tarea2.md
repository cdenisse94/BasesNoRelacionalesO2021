Actividad Integradora
========================================

1. ¿Qué idiomas base son los que más tuitean con hashtags? ¿Cuál con URLs? ¿Y con @?   
	Con Hashtags
	```javascript
	db.tweets.aggregate([
		{$lookup: {from:"primarydialects","localField":"user.lang","foreignField":"lang","as":"language"}},
		{$lookup: {from:"languagenames","localField":"language.locale","foreignField":"locale","as":"fulllocale"}},
		{$match:{"entities.hashtags":{$not:{$size:0}}}},
		{$group: {_id:"$fulllocale.languages", "conteo": {$count:{}}}}
	])
	```
	Con URLs
	```javascript
	db.tweets.aggregate([
		{$lookup: {from:"primarydialects","localField":"user.lang","foreignField":"lang","as":"language"}},
		{$lookup: {from:"languagenames","localField":"language.locale","foreignField":"locale","as":"fulllocale"}},
		{$match:{"entities.urls":{$not:{$size:0}}}},
		{$group: {_id:"$fulllocale.languages", "conteo": {$count:{}}}}
	])
	```
	Con User Mentions
	```javascript
	db.tweets.aggregate([
		{$lookup: {from:"primarydialects","localField":"_id","foreignField":"lang","as":"language"}},
		{$lookup: {from:"languagenames","localField":"language.locale","foreignField":"locale","as":"fulllocale"}},
		{$match:{"entities.user_mentions":{$not:{$size:0}}}},
		{$group: {_id:"$user.lang", "conteo": {$count:{}}}}
	])
	```

2. Qué idioma base es el que más hashtags usa en sus tuits?   
	```javascript
	db.tweets.aggregate([
		{$lookup: {from:"primarydialects","localField":"_id","foreignField":"lang","as":"language"}},
		{$lookup: {from:"languagenames","localField":"language.locale","foreignField":"locale","as":"fulllocale"}},
		{$group: {_id:"$user.lang", "totalHashtags": {$sum:{$size:"$entities.hashtags"}}}},
		{$project:{"language":0}},
		{$sort:{"totalHashtags":-1}}
	])
	```

Tarea 02
========================================

4. ¿Cómo podemos saber si los tuiteros hispanohablantes interactúan más en las noches?
   
   Consideramos noche entre las 7pm y la media noche
		
	*Tweets hechos por hispanohablantes entre las 19:00:00 y 23:59:59*
	```javascript
	db.tweets.aggregate([
		{$lookup: {from:"primarydialects","localField":"user.lang","foreignField":"lang","as":"language"}},
		{$lookup: {from:"languagenames","localField":"language.locale","foreignField":"locale","as":"fulllocale"}},
		{$match:{"user.lang":'es',"created_at":/([1]+[9]|[2]+[0-3])+:+([0-5]|[0-9])+:+[0-5]+[0-9]/}},
		{$group: {_id:"$fulllocale.languages", "conteo": {$count:{}}}}
	])
	```

	*Tweets hechos por hispanohablantes el resto del día*
	```javascript
	db.tweets.aggregate([
		{$lookup: {from:"primarydialects","localField":"user.lang","foreignField":"lang","as":"language"}},
		{$lookup: {from:"languagenames","localField":"language.locale","foreignField":"locale","as":"fulllocale"}},
		{$match:{"user.lang":'es',"created_at":/([0]+[1-9]|[1]+[0-8])+:+[0-5]+[0-9]+:+[0-5]+[0-9]/}},
		{$group: {_id:"$fulllocale.languages", "conteo": {$count:{}}}}
	])
	```
	Los tweets que obtenemos de los hispanoblantes son **1407** vs los **978** que suceden el resto del día (*transcurso de 19 horas*), por lo que se puede saber que **si** interactuan más en las noches

5. ¿Cómo podemos saber de dónde son los tuiteros que más tiempo tienen en la plataforma?
	
	*Base para dar orden a los meses*
	``` javascript
	db.months.insertMany([
		{month:"Jan", order:"01"},
		{month:"Feb", order:"02"},
		{month:"Mar", order:"03"},
		{month:"Apr", order:"04"},
		{month:"May", order:"05"},
		{month:"Jun", order:"06"},
		{month:"Jul", order:"07"},
		{month:"Aug", order:"08"},
		{month:"Sep", order:"09"},
		{month:"Oct", order:"10"},
		{month:"Nov", order:"11"},
		{month:"Dec", order:"12"}
	])
	```

	*Los 5 que más tienen en la plataforma mediante su fecha de creación de usuario*
	``` javascript
	db.tweets.aggregate([
		{$project:{"month": {$substr: ["$user.created_at",4,3]}, "day": {$substr: ["$user.created_at",8,2]}, "year": {$substr: ["$user.created_at",26,4]}, "user.screen_name":1}},
		{$lookup:{from:"months",localField:"month",foreignField:"month", as: "order"}},
		{$unwind:"$order"},
		{$project:{"date":{$concat:["$year","-","$order.order","-","$day"]}, "user.screen_name":1, "user.time_zone":1 }},  
		{$sort:{"date":1}},
		{$project:{"_id": 0, "user.screen_name":1, "date":1}},
		{$limit:5}
	])
	```

6. En intervalos de 7:00:00pm a 6:59:59am y de 7:00:00am a 6:59:59pm ¿De qué país es la mayoría de los tuits?
	
	*Tweets hechos de 7am a 7pm - Schedule1*
	```javascript
	db.tweets.aggregate([
		{$project:{"time":{$substr: ["$created_at",11,8]}, "place":"$user.time_zone"}},
		{$addFields:{"schedule":{$cond:
			{if: {$and: [{$gte: [{$toInt: {$substr:["$time",0,2]}},7]}, {$lte: [{$toInt: {$substr: ["$time",0,2]}}, 18]}]}
			,then: "7am-7pm", 
			else:"7pm-7am"}}, }},
		{$match:{"schedule":"7am-7pm"}},
		{$group:{_id: "$place", "conteo":{$count:{}}}},
		{$sort:{"conteo":-1}},
		{$limit: 2}
	])
	``` 

	*Tweets hechos de 7pm a 7am - Schedule2*
	```javascript
	db.tweets.aggregate([
		{$project:{"time":{$substr: ["$created_at",11,8]}, "place":"$user.time_zone"}},
		{$addFields:{"schedule":{$cond:
			{if: {$and: [{$gte: [{$toInt: {$substr:["$time",0,2]}},7]}, {$lte: [{$toInt: {$substr: ["$time",0,2]}}, 18]}]}
			,then: "7am-7pm", 
			else:"7pm-7am"}}, }},
		{$match:{"schedule":"7pm-7am"}},
		{$group:{_id: "$place", "conteo":{$count:{}}}},
		{$sort:{"conteo":-1}},
		{$limit: 2}
	])
	``` 

7. ¿De qué país son los tuiteros más famosos de nuestra colección?

	*Consideramos a los 3 más famosos*
	```javascript
	db.tweets.aggregate([
		{$project: {"user.followers_count":1, "user.lang":1}},
		{$lookup: {from:"primarydialects", "localField":"user.lang", "foreignField":"lang", "as":"language"}},
		{$lookup: {from:"languagenames", "localField":"language.locale", "foreignField":"locale", "as":"fulllocale"}},
		{$project: {_id:0, "fulllocale.languages":1, "user.followers_count":1}},
		{$sort: {"user.followers_count":-1}},
		{$limit: 3}
	])
	```