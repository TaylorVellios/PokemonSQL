# Purpose
To create a local relational database of Pokemon data using Python-BS4 and Pandas for ETL in coordination with SQLAlchemy and Psycopg2 to communicate with pgAdmin.</br>

This point of this project is to take a look at how much data manipulation is necessary to create a functioning relational system with Pokemon data.</br>
Finally, where a relational system isn't viable, what should be used?

## Dependencies
| Python Library | Install |
|------------|---------|
|[Requests](https://pypi.org/project/requests/)|pip install requests|
|[BS4 - BeautifulSoup](https://pypi.org/project/bs4/)|pip install bs4|
|[Pandas](https://pandas.pydata.org/)|pip install pandas|
|[SQLAlchemy](https://www.sqlalchemy.org/)|pip install SQLAlchemy|
|[Psycopg2](https://pypi.org/project/psycopg2/)|pip install psycopg2-binary|

[pgAdmin](https://www.pgadmin.org/download/) - PostgreSQL

# Disclaimer
All Data for this project is scraped from [Serebii.net](https://www.serebii.net/) and [Pokemondb.net](https://pokemondb.net/).</br>
All python code within this repository is for educational purposes only.</br>
Anyone looking to build their own local Pokemon database, please see [The RESTful Pokemon API](https://pokeapi.co/) via [Pokedex.py](https://pypi.org/project/pokedex.py/)</br>


## Extracting, Transforming Character Data
If you are unfamiliar with the Pokemon franchise, each monster character is a goldmine of data.</br>
For the sake of organization, we will be creating two main tables separated by game function:
* Pokemon_Stats to contain Battle-related data
* Pokemon_Details to contain character-specific information (height, weight, breeding stats, etc)

As you will see in a moment, we will be making a third table, 'Abilities', that will assist us with cleaning up the raw scrape in Pokemon_Stats.<br></br>

## Pokemon_Stats
Based on scraping [two](https://www.serebii.net/pokemon/all.shtml) [pages](https://pokemon.fandom.com/wiki/List_of_Pok%C3%A9mon_by_evolution), the first DataFrame being made for battle-related character data comes out pretty raw even after 50+ lines of code.</br>

![stats_raw](https://user-images.githubusercontent.com/14188580/118860333-e0c1af80-b8a0-11eb-82ec-c5c8f4ee7b0a.PNG)

Cleaning:</br>
At this point, the 'Type' and 'Abilities' columns are not suitable for a postgreSQL database and will need exploding into new columns.
* 'Type' is a simple fix with list comprehension, there can only be a maximum of two types per Pokemon, and each type is only one word.
* 'Abilities' on the other hand, can have up to 4 per Pokemon, and can be up to 3 words in length.

Before locking this table into a .csv we will have to retrieve all of the possible Abilities to cross reference against this messy column.<br></br>

## Abilities
Based on [this](https://pokemondb.net/ability) page, obtaining all of the possible Abilities was the easiest scrape in this project.</br>

![abilities](https://user-images.githubusercontent.com/14188580/118676935-71c15980-b7c1-11eb-963e-a0307eb8a185.PNG)

Using the list of abilities, we can then use itertools to make some sense out of the Abilities column in the pokemon_stats table.</br>
```
for i in pokemon['Abilities']:
    no1 = i.split(' ')
    no2 = [' '.join(f) for f in permutations(i.split(), r=2)]
    no3 = [' '.join(f) for f in permutations(i.split(), r=3)]
```
Above, I'm using the permutations function imported from itertools to generate three lists of possible string variations to check for valid abilities.</br>
* no1 is a list of all single words found for the current ability cell
* no2 is all combinations of strings within the current cell that could be made with 2 words (r=2)
* no3 is the same as no2 but attempting to make permutations with a word length of 3 (r=3)

After looping through the dataframe and parsing the abilities out to 4 new columns, we have our first completed table:</br>

![stats_final](https://user-images.githubusercontent.com/14188580/118862531-752d1180-b8a3-11eb-9835-0336f8716b2a.PNG)

## Pokemon_Details
Based on [this](https://pokemondb.net/pokedex/national) page, the Details table requires requesting and parsing a new page for every Pokemon.</br>
Due to the amount of pings made to the host server, I made sure to include a 1 second time.sleep() function per iteration of Pokemon name.</br>

![details_raw](https://user-images.githubusercontent.com/14188580/118681840-856ebf00-b7c5-11eb-8584-b8a226e7ec93.PNG)

I learned the hard way going into this section that there are some Pokemon that have names that don't have a 1:1 representation between their name and PokemonDB url.
</br>
The following dictionary is the result of playing some url golf.</br>
```
to_change = {'Mr. Mime': 'mr-mime',
             'Mime Jr.': 'mime-jr',
             'Flabébé': 'flabebe',
             'Type: Null': 'type-null',
             'Tapu Koko': 'tapu-koko',
             'Tapu Lele': 'tapu-lele',
             'Tapu Bulu': 'tapu-bulu',
             'Tapu Fini': 'tapu-fini',
             "Sirfetch'd": 'sirfetchd',
             'Mr. Rime': 'mr-rime'}
```
The main loop for this table accomplishes two things:
* obtaining as much data as possible from each page for the dataframe
* saving the official artwork for each character under a new folder in this repo's directory named 'Artwork'

![imgs](https://user-images.githubusercontent.com/14188580/118687036-32e3d180-b7ca-11eb-8076-40dd0d350d8c.PNG)

While these images don't help much with our goal of importing data into a postgres database, it is a very handy bit of code to keep in the event we decide to build a proper JSON object later on down the road. Since we are already hitting a page for every single pokemon, might as well grab what we can while we are there.</br>

The only columns that DON'T need cleaning in the raw details table are 'Name' and 'Species'.</br>
Using Regex, Pandas functions, list comprehensions, and a large looping section to handle the EVs_Given, the following table is our final output for the Pokemon_Details table:</br>
```
pokemon_details['Egg_Steps'] = pokemon_details['Egg_Steps'].map(lambda x: re.findall('\-(.+)\s',x)[0].replace(',',''))
pokemon_details['Catch_Rate'] = pokemon_details['Catch_Rate'].apply(lambda x: x.split()[0])
pokemon_details['Base_Friendship'] = pokemon_details['Base_Friendship'].apply(lambda x: x.split(' ')[0])

pokemon_details['Height'] = pokemon_details['Height'].map(lambda x: re.findall('\((.*)\)', x)[0])
pokemon_details['Weight'] = pokemon_details['Weight'].map(lambda x: re.findall('\((.*)lbs\)', x)[0])
pokemon_details = pokemon_details.rename(columns={'Weight':'Weight_lbs'})

genders = [(i.split(',')[0].replace('% male',''), i.split(',')[1].replace('% female','')) if i!='Genderless' else i for i in pokemon_details['Gender_Spread']]
pokemon_details['Male%'] = [i[0] if len(i)==2 else 'Genderless' for i in genders]
pokemon_details['Female%'] = [i[1] if len(i)==2 else 'Genderless' for i in genders]
pokemon_details = pokemon_details.drop(columns=['Gender_Spread'])

egg_groups = [[j for j in i.split() if not j.replace(',','').isnumeric()] for i in pokemon_details['Egg_Groups']]
pokemon_details['Egg_Group1'] = [i[0].replace(',','') for i in egg_groups]
pokemon_details['Egg_Group2'] = [i[1].replace(',','') if len(i)>1 else 'None' for i in egg_groups]
pokemon_details = pokemon_details.drop(columns=['Egg_Groups'])

```

![details_final](https://user-images.githubusercontent.com/14188580/118691884-21e98f00-b7cf-11eb-96fa-af04068869f7.PNG)

With the three dataframes that we have so far, the tables and their relationships look something like this:</br>

![diag](https://user-images.githubusercontent.com/14188580/118863233-3f3c5d00-b8a4-11eb-9b14-c787a3653f61.PNG)

As it currently stands, there is no good reason why we can't combine both Pokemon_Stats and Pokemon_Details into one large table other than for the sake of simplicity and exhibition.</br>

# PostgreSQL
Connecting to a local postgreSQL server in python is a fairly simple process.</br>
In the pokemonpostgres.ipynb file, you will see the steps taken to create a database from scratch with the tables built above.</br>

Be advised there is a config.py file with my personal database password saved as a string variable named *db_pass* that is being imported.</br>
Should you use this code as a jumping off point, the connection will fail without your password implemented appropriately for cells 2 and 3.</br>

After connecting to the local postgreSQL server with psycopg2, creating the database, and connecting to the new database with sqlalchemy, we can add the Pokemon .csv files to new tables.</br>
![db1](https://user-images.githubusercontent.com/14188580/119540997-b154ec00-bd53-11eb-923e-b28d6fe2ea8c.PNG)

While there are approaches within sqlalchemy to use dot notation with pythonic functions to query databases, I prefer to use the .execute() method to query with SQL commands.</br>
Below each cell where a table is created is a simple query to confirm our import was a success.</br>

Looking in pgAdmin, after a refresh of our local server we can confirm the new database and its tables:
![postgr1](https://user-images.githubusercontent.com/14188580/119545886-e9126280-bd58-11eb-96cf-3e9825c80529.PNG)
</br>

