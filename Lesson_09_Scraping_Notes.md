# Scraping Lecture 

## Lecture Notes

### Table of contents

These lecture notes are quite long, and not everything was covered during the lecture. Therefor a table contents, what you can expect in these notes:

- [Getting Public Data from the Government](#getting_public_data_from_the_government)
	- (A short section with some links and remarks from the investigation into special trees in Arnhem)
- [Scraping Wikipedia](#scraping_wikipedia)
	- (Introduction)
- [Getting the cities](#getting_the_cities)
- [Dive into cities](#dive_into_cities)
	- [Find the tables](#find_the_tables)
	- [No tables, just (un)ordered lists](#no_tables,_just_(un)ordered_lists)
	- [Combine the solution for tables and lists in one script](#combine_the_solution_for_tables_and_lists_in_one_script)
	- [Add a progress bar](#add_a_progress_bar)
- [Getting Coordinates](#getting_coordinates)


## Getting Public Data from the Government

##### 1. Waardevolle bomen

http://geo1.arnhem.nl/arcgis/services/Openbaar/Waardevolle_bomen/MapServer/WFSServer?request=GetFeature&service=WFS

`> TypeName is mandatory if featureID isn't present in GET requests.`

- python OWSLib?
    - http://geopython.github.io/OWSLib/

Or construct the url yourself

##### 2. Contruct the url

http://geo1.arnhem.nl/arcgis/services/Openbaar/Waardevolle_bomen/MapServer/WFSServer?request=GetFeature&service=WFS&TypeName=Openbaar_Waardevolle_bomen:Waardevolle_bomen

##### 3. Map geo points

What is this?

https://en.wikipedia.org/wiki/Geography_Markup_Language

SRS = Spacial Reference System

`<gml:Envelope srsName="urn:ogc:def:crs:EPSG:6.9:28992">`

- https://www.epsg-registry.org
    - is a type of srs: namely EPSG::28992 = "Amersfoort / RD New"
- How to convert?
    - http://cs2cs.mygeodata.eu
- or with python:
    - http://gis.stackexchange.com/questions/78838/how-to-convert-projected-coordinates-to-lat-lon-using-python

----

## Scraping
### - from data to graph -

We have a better title:

## Scraping Wikipedia
### - Debugging the variability of Wikipedia -

##### Apps open:

- Textedit for notes
- Terminal window with tabs for git / ipython / executing
- Browser
- Textmate for editing

### Starting point:

- Wikipedia page:
    - https://en.wikipedia.org/wiki/Lists_of_cities_by_country




## Getting the cities

#### Attempt 1: wikipedia module

- python module: *wikipedia*
    - [Quick Start](https://wikipedia.readthedocs.org/en/latest/quickstart.html#quickstart)
    - [Full Documentation](https://wikipedia.readthedocs.org/en/latest/code.html#api)

```
import wikipedia
wikipedia.page("Lists_of_cities_by_country")
p = wikipedia.page("Lists_of_cities_by_country")
p.sections
p.categories
p.links
```

(`p.links` returns a list, of *all* links, alphabetically ... not what we want)  
So let's try to scrape the list by parsing the DOM tree (with Beautiful Soup)

#### Attempt 2: parse the DOM

```
from bs4 import BeautifulSoup
import urllib
r = urllib.urlopen("https://en.wikipedia.org/wiki/Lists_of_cities_by_country").read()
soup = BeautifulSoup(r)
type(soup)
soup.findAll("li")
listitems = soup.findAll("li")
```

`len(listitems)`

`> 353`

These are too much list items

So we need to find another way to get the prefered list items, ... maybe use the flag:

Find all flags, then get the (parent) item that holds that flag, that parent item will probably contain the two links: the list of cities PLUS the name of the country

```
flags = soup.find_all("span", "flagicon")
flags = soup.find_all("span", class_="flagicon")
len(flags)
listitems = []
for f in flags:
    listitems.append(f.parent)
len(listitems)
listitems[0]
```

Check if we have all the countries by printing just the countries

```
l = listitems[0]
l.findChildren("a")
len(l.findChildren("a"))
l.findChildren("a")[-1]
l.findChildren("a")[-1]['title']
l.findChildren("a")[-1].text
```

```
for l in listitems:
    print l.findChildren("a")[-1]['title']
```

Fine!

Finally check if the list is sane, by counting the number of links of each item:

```
for l in listitems:
   print len(l.findChildren("a")), l.findChildren("a")[-1]['title']
```
   
Oh, there is a problem with Vatican City. ... Let's skip that one by ensuring the len( ... "a") is bigger or equal to 3.
And there is a problem with Ireland, because it has two flags. (Let's leave that in for now)


Now, list find all the "List of cities in ..." pages, and store them together with the country, in list, like this:
```
[
	{'citiesList': "List of cities in Brazil", 'country': "Brazil"},
	{...},
	...
]
```

```
citiesData = []
for l in listitems:
    if len(l.findChildren("a")) >= 3:
        country = {}
        country['citiesList'] = l.findChildren("a")[-2]['title']
        country['country'] = l.findChildren("a")[-1]['title']
        citiesData.append(country)
```

As a step in between, lets save this list.

```
import json
with open("countries.json", 'w') as outputFile:
   json.dump(citiesData, outputFile)
```

Let's create a python script from this:

`history`

----

## Dive into cities

Now, let's continue and dive in the pages listing the cities. ...

There are many different types of pages, using tables, using lists, multiple tables, duplicates, historic names ... This will be lots of stuff we don't want. What would be the best approach?

My attempt will be the following:

1. searching the DOM tree (with Beautiful Soup):
2. if there are tables, use the first column of the table which has a link with a title
3. if there are no tables, use all li (listitems)
4. only store the city if it is not a duplicate

### Find the tables

First let's see if we can find the tables

```
c = citiesData[0]
p = wikipedia.page(c['citiesList'])
p.url
r = urllib.urlopen(p.url).read()
soup = BeautifulSoup(r)
tables = soup.findAll('table', class_="wikitable")
len(tables)
```

`> 2`

Ok. Let's try this with the first table we find

```
t = tables[0]
```

Get all rows:

```
t.findAll('tr')
```

try with the first, and then second row

```
tr = t.findAll('tr')[0]
tr
tr = t.findAll('tr')[1]
tr
```

find the first td in a row

```
td = tr.findAll('td')[0]
```

Finally, get the a.

```
td.find('a')
```

#### Bake it into the python script

Now let's put this together, and try it out for Afganistan

```
c = citiesData[0]
```

```
cities = []
```

```
for t in tables:
    for tr in t.findAll('tr'):
        for td in tr.findAll('td'):
            link = td.find('a')
            if link:
                cityName = link['title']
                if cityName:
                    # ...
                    cities.append(cityName)
```
				
`> Error`

What is link?  
` <a href="#cite_note-2">[2]</a>`
 
It has no attribute title, so it is not good

`link.has_attr('title')`

`> False`

Change the code:

```
for t in tables:
    for tr in t.findAll('tr'):
        for td in tr.findAll('td'):
            link = td.find('a')
            if link and link.has_attr('title'):
                cityName = link['title']
                if cityName:
                    # ...
                    cities.append(cityName)
```

#### Remove missing pages

Almost good. It still finds duplicates, and it also contains:  
'Laghman, Jowzjan (page does not exist)'

Let's exclude the pages that do not exist:

`cityName.endswith('(page does not exist)')`

`> True`

```
for t in tables:
    for tr in t.findAll('tr'):
        for td in tr.findAll('td'):
            link = td.find('a')
            if link and link.has_attr('title'):
                cityName = link['title']
                if cityName and not cityName.endswith('(page does not exist)'):
                    # ...
                    cities.append(cityName)
```

#### Remove duplicates

Now, let's remove the duplicates:

`'Kabul' in cities`

`> True`

`'New York' in cities`

`> False`

Expand the `# ...`

```
for t in tables:
    for tr in t.findAll('tr'):
        for td in tr.findAll('td'):
            link = td.find('a')
            if link and link.has_attr('title'):
                cityName = link['title']
                # remove duplicates and broken pages
                if cityName and not cityName.endswith('(page does not exist)') and not cityName in cities:
                    cities.append(cityName)
```

----

#### Only match the first valid link in a row 


All fine and well, but what if the row, contains more valid links?

Try with Algeria

```
c = citiesData[2] 
p = wikipedia.page(c['citiesList'])
p.url
r = urllib.urlopen(p.url).read()
soup = BeautifulSoup(r)
tables = soup.findAll('table', class_="wikitable")
len(tables)
```

(Same block code as before)

So, again, modify it a bit (add the break)

```
for t in tables:
    for tr in t.findAll('tr'):
        for td in tr.findAll('td'):
            link = td.find('a')
            if link and link.has_attr('title'):
                cityName = link['title']
                # remove duplicates and broken pages
                if cityName and not cityName.endswith('(page does not exist)') and not cityName in cities:
                    cities.append(cityName)
                    break
```

----

### No tables, just (un)ordered lists

Now let's try this when there are no tables:

Trying this out with: List of cities in Antigua and Barbuda

```
c = citiesData[5]
p = wikipedia.page(c['citiesList'])
r = urllib.urlopen(p.url).read()
soup = BeautifulSoup(r)
tables = soup.findAll('table', class_="wikitable")
len(tables)
```

First, what if we would just get all the list items?

```
lis = soup.findAll('li')
len(lis)
```

`> 147`

Clearly, that's way too much. We need to filter it down.

Maybe: only get the list items inside the mw-content-text div

```
div = soup.find('div', id="mw-content-text")
```

Note: find just returns the first object found, where findAll returns a list [] of objects.

`len(div.findAll('li'))`

`> 79`

That is still to much. Let's narrow it down further. There is a difference between all descendents and just the first level children:


`len(div.findAll('ul'))`

`> 5`

`len(div.findAll('ul', recursive=False))`

`> 1`

We need to be looking or ul and ol (unordered and ordered lists)

```
uls = div.findAll('ul', recursive=False)
```

Test with only the first

```
ul = uls[0]
ul.findAll('a')
```

```
ols = div.findAll('ol', recursive=False)
```

Test with only the first

```
ol = ols[0]
ol.findAll('a')
```

----

### Combine the solution for tables and lists in one script

Let's put this **ALL** together:

But let's limit it to the first 8 for now. Later, we'll remove the `[:8]`

```
for c in citiesData[:8]:
    p = wikipedia.page(c['citiesList'])
    p.url
    r = urllib.urlopen(p.url).read()
    soup = BeautifulSoup(r)
    tables = soup.findAll('table', class_="wikitable")
    # create an empty cities list
    cities = []
    if len(tables) > 0:
        # First the case when there are tables
        #
        for t in tables:
            for tr in t.findAll('tr'):
                for td in tr.findAll('td'):
                    link = td.find('a')
                    if link and link.has_attr('title'):
                        cityName = link['title']
                        # remove duplicates and broken pages
                        if cityName and not cityName.endswith('(page does not exist)') and not cityName in cities:
                            cities.append(cityName)
                            break
    else:
        # Find all valid links in list items
        div = soup.find('div', id="mw-content-text")
        # Search all unordered lists
        for ul in div.findAll('ul', recursive=False):
            for link in ul.findAll('a'):
                if link.has_attr('title'):
                    cityName = link['title']
                    # remove duplicates and broken pages
                    if cityName and not cityName.endswith('(page does not exist)') and not cityName in cities:
                        cities.append(cityName)
        # Search all ordered lists        
        for ol in div.findAll('ol', recursive=False):
            for link in ol.findAll('a'):
                if link.has_attr('title'):
                    cityName = link['title']
                    # remove duplicates and broken pages
                    if cityName and not cityName.endswith('(page does not exist)') and not cityName in cities:
                        cities.append(cityName)
    # Now, our cities list is filled with cities, but what to do with it.
    #
    # Let's append it to the dictionary c, we already had
    c['cities'] = cities
```

Excellent!

Let's put this in a file, and run it from there.

----

### Add a progress bar

You see that just getting the cities from the first 8 countries, already takes quite an amount of time. If we're going to run it on the whole list, we want to see some feedback during the process.

Install the tqdm module
https://github.com/tqdm/tqdm

```
sudo easy_install tqdm
```

```
from tqdm import tqdm
```

```
for i in tqdm(range(10)):
    # do something
```
    
----

Now, let's try it for all countries!

This will take approx. 5 minutes

----

There are still a few problems:

`cat cities.json | grep "page does"`

`> "citiesList": "List of cities in Somaliland (page does not exist)"`


So also we need to filter out the non-existing pages with list of cities. To bad for Somaliland, but we'll have to ignore it.

----

Also, there is still a problem with:

- Kenya
- Netherlands
- Northern Cyprus
- Pakistan
- Somalia

Let's at least fix the case for the Netherlands.

We do this by not only looking for the first `td` with a valid link, but also a `th` with a valid link. This will give more noise, but also include the Dutch cities.

```
		allLinks = []
		for th in tr.findAll('th'):
			link = th.find('a')
			if link:
				allLinks.append(link)
		for td in tr.findAll('td'):
			link = td.find('a')
			if link:
				allLinks.append(link)
```

----

## Getting Coordinates

Next up, is verifying a city really is a city, ... getting the lat-lon coordinates and getting the population ...

First try it with a simple "city" or not city:

- "Kabul"
- "Amsterdam"
- "Geographic coordinate system"

```
p = wikipedia.page("Kabul")
p.coordinates
```

```
p = wikipedia.page("Amsterdam")
p.coordinates
```

```
p = wikipedia.page("Geographic coordinate system")
p.coordinates
```

`> KeyError`

So how can we make sure we don't get this error? We don't know if the page has a coordinate or not...

#### Option 1: We check for a category

```
p = wikipedia.page("Amsterdam")
p.categories
```

It contains the category "Coordinates on Wikidata"

```
p = wikipedia.page("Geographic coordinate system")
p.categories
```

It doesn't contain that category.

How do we check for that? With the `in` keyword

```
"Coordinates on Wikidata" in p.categories
```

`> False`

And when there are coordinates:

```
p = wikipedia.page("Kabul")
"Coordinates on Wikidata" in p.categories
```

`> True`

#### Option 2: We can also use the KeyError as an `Exception`

```
try:
    # do something what could trigger an error
except Error:
    # catch the error
```

In this case:

```
try:
    coord = p.coordinates
except KeyError:
    print "no coordinates"
```

This is a safe, simple and quick solution. So let's build this second option into the code we already have for finding the cities!

----

### Build the coordinates finding in our script

We should change some things first:

1. Not only store a list of cities (`cities = []`), where each element is a string, but also a list of `cityDicts` [{}, {}, ...], where each element is a dict, with more information about the city.
	- 1b. We keep the original list, to skip duplicates as before.
2. Refactor some code, because we're now doing the same thing 4 times
3. This will take considerably longer than before:
	- So we'll limit the list to only the first 8 countries


As the first thing to do, we start with the refactor, so the rest of the work will be easier:

#### 2. Refactor into appendLinkToCities() function

```
def appendLinkToCities(link, cityNameList):
	if link and link.has_attr('title'):
		cityName = link['title']
		# remove duplicates and broken pages
		if cityName and not cityName.endswith('(page does not exist)') and not cityName in cityNameList:
			cityNameList.append(cityName)
			return True
	return False
```

#### 3. Test it with a limited set of countries

```
for c in citiesData[:8]:
```

#### 1. Create a cityDicts

- Create it along the cities list
- Send it along with the cities to the function
- Use it instead of the cities list to store the data in the citiesData
- In the function, append a dictionary to cityDicts

##### Create it along the cities list

```
		cityDicts = []
```

##### Send it along with the cities to the function

```
		for link in allLinks:
			if appendLinkToCities(link, cities, cityDicts):
```

##### Use it instead of the cities list to store the data in the citiesData

```
	numberOfCities += len(cityDicts)
	c['cities'] = cityDicts
```

##### In the function, append a dictionary to cityDicts

```
def appendLinkToCities(link, cityNameList, cityDictList):
...
				cityDictList.append(cityDict)
				cityNameList.append(cityName)
```

----

Now we're ready to add the geolocation to the cities

```
coord[0]
coord[1]
float(coord[0])
```

Try this with a very limited set [:2]
And let's also store the url ... could be useful later.

```
				cityDict['name'] = cityName
				cityDict['lat'] = float(coord[0])
				cityDict['lon'] = float(coord[1])
				cityDict['url'] = p.url
```

----

#### Add a second progress bar

Wow, this really takes a lot of time ... Let's add another progress bar.

How to do this? We have several for loops, finding links. ...

Let's refactor: 

- First find all possible links (all anchors)
- Then process these anchors and add the successful ones

possibleCityLinks will be a list of lists [[]]

----

### Let's enter them BUGS!

Now, that that's done. Let try this for all countries!

...

Ooooh... This will take 1 and a half hour!

Ooooooh - 2 ... and we hit a bug:

```
Traceback (most recent call last):
  File "scrape_wiki_countries.py", line 122, in <module>
    main()
  File "scrape_wiki_countries.py", line 108, in main
    if appendLinkToCities(link, cities, cityDicts):
  File "scrape_wiki_countries.py", line 17, in appendLinkToCities
    p = wikipedia.page(cityName)
  File "build/bdist.macosx-10.10-intel/egg/wikipedia/wikipedia.py", line 276, in page
  File "build/bdist.macosx-10.10-intel/egg/wikipedia/wikipedia.py", line 299, in __init__
  File "build/bdist.macosx-10.10-intel/egg/wikipedia/wikipedia.py", line 345, in __load
wikipedia.exceptions.PageError: Page id "catolica" does not match any pages. Try another id!
```

`PageError?`

Do not understand ... but hey, let's try solve it. Let's make an exception for this error.

Then, later we get another error:

`DisambiguationError`

What's going on here?

Maybe it is trying to find the wrong pages when we do:

```
	p = wikipedia.page(cityName)
```

Let's read the documentation again, and discover that `auto_suggest` is `True` by default. We might not want that.  

(Auto_suggest means that the text we give the page function with `page(...)`, will be first checked for possible other pages with a similar name. So now Wikipedia is trying to play smart on us, where we exactly know which page we would like to have. It better not do that, and just give us the requested page straight up. The `PageError` problem with "catolica" was also due to these same smart suggestions.)

```
	p = wikipedia.page(cityName, auto_suggest=False)
```
    
Hey, now it is faster too!

----

Snag! Another DisambiguationError with the City _Aw_ in Bahrain
Guess we need the `try:` anyway

----

#### Exclude list-of-cities-pages which aren't list-of-cities, but just the country

Darn... Another bug:

```
Processing Denmark:  21%|███████████████████████████████▉
  File "scrape_wiki_countries.py", line 125, in <module>
    main()
  File "scrape_wiki_countries.py", line 111, in main
    if appendLinkToCities(link, cities, cityDicts):
  File "scrape_wiki_countries.py", line 18, in appendLinkToCities
    p = wikipedia.page(cityName, auto_suggest=False)
  File "build/bdist.macosx-10.10-intel/egg/wikipedia/wikipedia.py", line 276, in page
  File "build/bdist.macosx-10.10-intel/egg/wikipedia/wikipedia.py", line 299, in __init__
  File "build/bdist.macosx-10.10-intel/egg/wikipedia/wikipedia.py", line 398, in __load
KeyError: u'fullurl'
```

So, there is another problem. This time with *Akrotiri and Dhekelia*

And on closer inspection:
- Gibraltar
- Monaco
- Montserrat
- Pitcairn Islands
- South Ossetia
- Svalbard				(Spitsbergen)
- Transnistria

All these countries do not have a list of cities page (it is the same as the country).
So we need to check that.

```
			if not listPage.endswith('(page does not exist)') and not listPage == countryName:
```

And another exception with Aswan in Egypt. Let's combine all the exceptions.

... this will take about 2,5 hours and result in:

Found list of 227 countries
Found list of 16820 cities

----

#### More problems with the way Wikipedia has its data structured

Other problems:

- Dominican Republic:
	not the correct list of cities page (need to go one step further)
- United States:
	not the correct list of cities page (need to go one step further)

- Czech Republic:
	flags in the table as first anchor (without title)


Missing the United States feels like a problem we really need to fix first.
So let's look at the page, ... it lists more pages, ... each one of them would be fine,
but "List of United States cities by population" would be ideal.

How to best fix this? 

Let's split up our efforts into seperate scripts. Not a "clean" solution, but a very pragmatic one. And it will also help us speed the process.

Make the scrape_wiki_countries only scrape countries and deliver a file countries.json 
Then use the countries.json to be used in a new script scrape_wiki_cities and deliver a second file cities.json.

----

##### Step 1: scrape the countries

First step: scrape just the countries:

`python scrape_wiki_countries.py`

##### Step 2:

Then manually fix the countries.json, specifically:

- Dominican Republic
- United States

---- 

##### Step 3: scrape the cities

Next is our scrape_wiki_cities script.

- We read in the json file
- Checking for listPage == countryPage, is already done in the first script, so we just need to check for an empty string

```
test = ""
if test:
    print "not here"
else:
    print "but here"
test = "some string"
```

`if listPage:`

----

## Get the population

Now we have about 17343 cities, of which we know the latitute and longitude. Nice!
We could already map these cities on a map, ...

But I want more: what if we could get the population of all these 17343 cities?
The wikipedia pages know this, so let's get this out.

Let's investigate the population of a few cities:


```
from bs4 import BeautifulSoup
import urllib
```

```
url = "https://en.wikipedia.org/wiki/Kabul"
r = urllib.urlopen(url).read()
soup = BeautifulSoup(r, "html.parser")
```

```
infoBox = soup.find('table', class_="infobox")
```

```
mergedTopRows = infoBox.findAll('tr', class_="mergedtoprow")
len(mergedTopRows)
```

```
for m in mergedTopRows:
    print m.text
```
	
```
for m in mergedTopRows:
	text = m.text
    if text.startswith("Population"):
        print m
```
		
Nothing? Well, it turns out there is some white space before`

```
text = "    Hello    "
text
text.lstrip()
text.rstrip()
text.strip()
```

```
for m in mergedTopRows:
    text = m.text.strip()
    if text.startswith("Population"):
        print m
		merged = m		
        
merged
```

```
merged.findNextSiblings('tr', class_="mergedrow")
tr = merged.findNextSiblings('tr', class_="mergedrow")[0]
```

```
tr
tr.attrs
tr['class']
```

```
if 'mergedrow' in tr['class']:
    print tr
```
	
```
for tr in merged.findNextSiblings('tr'):
    if 'mergedrow' in tr['class']:
        print tr.find('th').text
    else:
        break
```

----

#### Put this all together

That was a lot. Let's put this all together.

##### 1. First create a new script that reads the cities.json

##### 2. The cities.json is a nested list. Make the code parse the data in the same nested way


First get all the cities

```
with open("cities.json", 'r') as inputFile:
   citiesData = json.load(inputFile)
```

The citiesData is a nested list of countries with cities

```
pBarCountries = tqdm(citiesData, leave=True, nested=True)
for c in pBarCountries:
	country = c['country']
	if c.has_key('cities'):
		cities = c['cities']
		pBarCountries.set_description("Processed %s" % country) # (Unfortunately set_description is only used after processing)
		pBarCities = tqdm(cities, leave=True, nested=True)
		for city in pBarCities:
			# Open the city page
```


##### 3. Open the page for the city

```
			# Open the city page
			r = urllib.urlopen(city['url']).read()
			soup = BeautifulSoup(r, "html.parser")
			# Get the infobox table
			infoBox = soup.find('table', class_="infobox")
			# Get all mergedTopRows
			mergedTopRows = infoBox.findAll('tr', class_="mergedtoprow")
```

##### 4. Find the population merged top row

```
			populationMergedTopRow = None
			for m in mergedTopRows:
			    text = m.text.strip()
			    if text.startswith("Population"):
					populationMergedTopRow = m
					break
			if populationMergedTopRow:
				# continue
```

##### 5. Find sibling tr's only with class 'mergedrow'. Break at the first non-"mergedrow"

```
			if populationMergedTopRow:
				for tr in populationMergedTopRow.findNextSiblings('tr'):
					if 'mergedrow' in tr['class']:
					print tr.find('th').text
				else:
					break
```

##### 6. Find the population key and population value
				
```
				for tr in populationMergedTopRow.findNextSiblings('tr'):
					if 'mergedrow' in tr['class']:
					populationKey = tr.find('th').text.strip()
					if populationKey:
						populationValue = tr.find('td').text.strip()
```
					
##### 7. If it has a key and a value, store it in a dict

```
			pDict = {}
			
					if populationKey:
						populationValue = tr.find('td').text.strip()
						if populationValue:
							# Add the key and value to a sub - dict
							pDict[populationKey] = populationValue
							# Keep track of whether population was found
							foundPopulation = True
```

##### 8. Store the dict with the city

```
			if foundPopulation:
				# increment a counter
				numberOfCities += 1
				# Add the info to the city data
				city['populationInfo'] = pDict
```
							
##### 9. And save into a new json

Finally: save a file, with all cities by countries

```
with open("city_populations.json", 'w') as outputFile:
   json.dump(citiesData, outputFile, indent=2)
print "Found %d cities with population" % (numberOfCities)
```

##### 10. Try it: 

It still contains all these weird chararcters. Let's strip them:

```
key = merged.findNextSibling().find('th').text
key
key.strip(u'\xa0\u2022\xa0')				
```

----

#### Just a few more bugs to solve

More bugs and more bugs:

```
Processing Argentina:   3%|████▋
Traceback (most recent call last):
  File "scrape_wiki_city_population.py", line 71, in <module>
    main()
  File "scrape_wiki_city_population.py", line 48, in main
    if 'mergedrow' in tr['class']:
  File "build/bdist.macosx-10.9-intel/egg/bs4/element.py", line 905, in __getitem__
KeyError: 'class'
```

Prevent this bug by adding this extra check:

`if tr.has_attr('class') and 'mergedrow' in tr['class']:`

....

And another bug:

```
Processing Azerbaijan:   5%|███████▏
Traceback (most recent call last):
  File "scrape_wiki_city_population.py", line 71, in <module>
    main()
  File "scrape_wiki_city_population.py", line 49, in main
    populationKey = tr.find('th').text.strip().strip(u'\xa0\u2022\xa0')
AttributeError: 'NoneType' object has no attribute 'text'
```

Prevent this bug by again, adding an extra check:

```
th = tr.find('th')
if th and th.text:
	populationKey = th.text.strip().strip(u'\xa0\u2022\xa0')
```

----

## Alternative approaches?

Are there alternative approaches to the problem we have? We want to get the all the cities in the world and their populations.

Could we find the cities by clever use of categories ...?

Starting point: https://en.wikipedia.org/wiki/Category:Cities_by_country

This might be an alternative way of doing things by using categories. Unfortately, just as you cannot rely on wikipedia having the same structure on different pages listing cities in a country, you can also not rely on wikipedia for each page to have all the correct categories. So this might still be an interesting approach, but for this lecture we keep by working with the list of list page.