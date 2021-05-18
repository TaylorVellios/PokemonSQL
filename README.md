# Purpose
To create a local relational database of Pokemon data using Python-BS4 and Pandas for ETL in coordination with SQLAlchemy and Psycopg2 to communicate with pgAdmin.</br>

This project is taking a look at how much data manipulation is necessary to create a functioning relational system with Pokemon data.</br>

### Dependencies
| Python Library | Install |
|------------|---------|
|[Requests](https://pypi.org/project/requests/)|pip install requests|
|[BS4 - BeautifulSoup](https://pypi.org/project/bs4/)|pip install bs4|
|[Pandas](https://pandas.pydata.org/)|pip install pandas|
|[SQLAlchemy](https://www.sqlalchemy.org/)|pip install SQLAlchemy|
|[Psycopg2](https://pypi.org/project/psycopg2/)|pip install psycopg2-binary|

|PostgreSQL Server|
|-----------------|
|[pgAdmin](https://www.pgadmin.org/)|

## Disclaimer
All Data for this project is scraped from [Serebii.net](https://www.serebii.net/) and [Pokemondb.net](https://pokemondb.net/).</br>
All python code within this repository is for educational purposes only.</br>
For anyone looking to build their own local Pokemon database, please see [The RESTful Pokemon API](https://pokeapi.co/) via [Pokedex.py](https://pypi.org/project/pokedex.py/)</br>
A Relational Database is not the ideal storage solution for this type of data, use the API above if you are interested in acquiring mass Pokemon data.</br>

## Extracting, Transforming Character Data
If you are unfamiliar with the Pokemon franchise, each monster character is a goldmine of data.</br>
For the sake of organization, we will be creating two main tables separated by game function:
* Pokemon_Stats to contain Battle-related data
* Pokemon_Details to contain character-specific information (height, weight, breeding stats, etc)

### Pokemon_Stats
Based on [this](https://www.serebii.net/pokemon/all.shtml) page, there are two main focus points that need addressing after we get a raw scrape.</br>

![stats_raw](https://user-images.githubusercontent.com/14188580/118673075-556fed80-b7be-11eb-8d55-6b4110fac5ad.PNG)

The 'Type' and 'Abilities' columns are both returned as strings that will need separating into their own columns.</br>
'Type' is a simple fix, there can only be a maximum of two types per Pokemon, and each type is only one word.</br>
'Abilities' on the other hand, can have up to 4 per Pokemon, and can be up to 3 words in length.</br>
Before locking this table into a .csv we will have to retrieve all of the possible Abilities to cross reference against this messy column.</br>

### Abilities
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
* no2 is all combinations of the current cell that could be made with 2 words (r=2)
* no3 is the same as no2 but attempting to make permutations with a word length of 3 (r=3)

After looping through the dataframe and parsing the abilities out to 4 new columns, we have our first completed table:</br>
![stats_final](https://user-images.githubusercontent.com/14188580/118679937-f1502800-b7c3-11eb-8d5b-bae9750abc1c.PNG)
