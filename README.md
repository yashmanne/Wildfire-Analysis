# Analysis of Wildfire Impact on Redmond, OR

The goal of this work is to explore the impact of wildfires on Redmond, Oregon. Specifically, we will be looking at the wildfire prevalence & air quality as a way to measure the smoke impacts over the past 60 years. A secondary goal of this work is to develop the reproducibility & professionalism skills required for real-world data-driven analysis as part of the Fall 2023 DATA 512 course at the University of Washington.

## Data Sources

### Wildfire Data
Cleaned, collated data of wildfires was generated by the US Geological Survey as the [Combined wildland fire datasets for the United States and certain territories, 1800s-Present (combined wildland fire polygons](https://www.sciencebase.gov/catalog/item/61aa537dd34eb622f699df81) dataset. For this project, we used the GeoJSON data format stored under `GeoJSON Files.zip`. This folder contains both a raw merged dataset containing duplicates and a "combined", duplicate-free, dataset that comprises both wildfires and prescribed fires from the mid-1800s to the 2021 collated from 40 different original wildfire datasets. For our analysis, we will use only the combined data stored at [`./input/USGS_Wildland_Fire_Combined_Dataset.json`](./input/USGS_Wildland_Fire_Combined_Dataset.json). Note that the data set is too large to track with Git and is therefore not available on this repository.

The data is listed as [public](https://www.sciencebase.gov/catalog/item/53f6271fe4b09d12e0e9bd03) and can be cited as the following:

Welty, J.L., and Jeffries, M.I., 2021, Combined wildland fire datasets for the United States and certain territories, 1800s-Present: U.S. Geological Survey data release, https://doi.org/10.5066/P9ZXGFY3.

It should be noted that much of this data was originally generated from an ArcGIS server and shares a lot of variable names with the ArcGIS software. For example, the reason the default geometry type is `esriGeometryPolygon` -- Esri is the developer of ArcGIS. As a result of this, some features, like accessing curved polygons stored in a format proprietary to ArcGIS proved difficult.

#### Data Description
The used GeoJSON file contained the following keys:
* `displayFieldName`: an empty string that would otherwise denote the name of the dataset.
* `fieldAliases`: a dictionary that converts the variable name in the file to a more human-readable format for the 30 fields.
* `geometryType`: `esriGeometryPolygon` is the default geometry format.
* [`spatialReference`](https://developers.arcgis.com/web-map-specification/objects/spatialReference/): the well-known ID (WKID) of the spatial reference (as well as the latest WKID). This data uses [ESRI:102008](https://epsg.io/102008), which refers to the Albers equal-area map projection of North America.
* `fields`: A list of dictionaries identifying the `name`, `type`, and `alias` of 30 attributes. The `alias` is the same as those in `fieldAliases`. Also, the `types` are based on the [Esri specifications](https://developers.arcgis.com/web-map-specification/objects/field/). A full, detailed explanation of each of the attributes can be found [here](./input/Wildland_Fire_Polygon_Metadata.xml).
* `features`: the list of all observations stored in JSON format. Note that each observation is saved as a dictionary with keys for the `attributes` (same as those in `fields`) and `geometry`, which contains sets of tuples denoting the coordinate points of the wildfire polygon in the WKID projection space. In the case of this data, most wildfires are polygons represented by ['rings'](https://developers.arcgis.com/documentation/common-data-types/geometry-objects.htm#:~:text=%7B%0A%20%20%22paths%22%3A%20%5B%20%5D%0A%7D-,Polygon,-A%20polygon%20(specified)) in ArcGIS. A ring is a list of points denoting the path of the ring. The exterior ring of a polygon is denoted by a list of points oriented clockwise while internal rings are points oriented in a counterclockwise motion. The `rings` key for `geometry` has multiple polygons decreasing in area. The first 'ring' denotes the largest boundary of the fire. Subsequent rings follow an even-odd fill rule (first fills, 2nd removes, etc). A few wildfires are represented by [`curveRings`](https://developers.arcgis.com/documentation/common-data-types/geometry-objects.htm#:~:text=y%5D%2C%20%5Bx%2C%20y%5D%5D%7D-,Polyline%20with%20curves,-A%20polyline%20with), which are similar to `rings` except that they define certain portions of a ring using parameters for preset curve functions rather than individual data points.

Of the 30 initial attributes, we select the following variables:
* `OBJECTID`: A unique ID for each fire polygon.
* `Assigned_Fire_Type`: The attributed type of the fire. Contains the following values: 'Wildfire', 'Likely Wildfire', 'Unknown - Likely Wildfire', 'Unknown - Likely Prescribed Fire', & 'Prescribed Fire'.
* `Fire_Year`: The year of the fire season (int).
* `GIS_Acres`: The overall area burned by the fire(s) in acres.
* `GIS_Hectares`: The overall area burned by the fire(s) in hectares.
* `Listed_Fire_Names`: A string of comma-separated values for the fire(s) attributed to the polygon.
* `Shape_Length`: A proprietary ESRI raw measurement of the longest section of the polygon in the ESRI projection space units (assumed m)
* `Shape_Area`: A proprietary ESRI raw measurement of the polygon area in the ESRI projection space units (assumed m^2).
* `rings`: A list of polygon rings defining the overall fire polygon, with the first one denoting the fire perimeter. 
* `curveRings`: A list of polygon curveRings defining the overall shape, with the first one denoting the fire perimeter.

We further subset the data on two main criteria:
  1. Fires from 1963 onwards (1963 inclusive).
  2. Fires within 1250 miles of Redmond, Oregon. 
The first is easy enough to filter. We do this to avoid potentially bad data estimated before the advent of satellite imaging. The second requires a couple of assumptions:
* We mark `(44.272621, -121.173920)` as the latitude & longitude coordinates as the center of the city. [Source](https://latitude.to/map/us/united-states/cities/redmond-oregon)
* We denote "within 1250 miles" to mean fires whose closest boundary point has a total straight-line ellipsoid distance less than 1250 miles to the center of the city. The first polygon ring is used as the boundary of the data. To calculate distance, we must convert the projection of the ring data from the current equal-area Albers projection ([`ESRI:102008`](https://epsg.io/102008)) to a more accurate, WGS84, decimal-degrees representation better for distance calculations ([`EPSG:4326`](https://epsg.io/4326)).

After filtering, we store the desired subset into [`./intermediate/redmond_fire_subset.csv`](./intermediate/redmond_fire_subset.csv). Note that the file remains too large to be tracked by Git but can be generated by the code in [`./analysis-part1-SmokeEstimates`](./analysis-part1-SmokeEstimates).

### Air Quality Index Data
A portion of this analysis required historical Air Quality Index (AQI) data for Redmond, Oregon, which is located in Deschutes County, during the fire season (May 1 - October 31) for each year from 1963 onwards. The Air Quality Index is a measure designed to tell us how healthy the air is on any given day and is commonly used to track pollutants such as smog or smoke. Generally, a rating of 0-50 indicates healthy, clean air, while 500 is the highest tracked value for hazardous air. A thorough explanation of how AQI is calculated can be found [here](https://www.airnow.gov/sites/default/files/2020-05/aqi-technical-assistance-document-sept2018.pdf).

In this project, we used the US Environmental Protection Agency (EPA) Air Quality Service (AQS) API. The [documentation](https://aqs.epa.gov/aqsweb/documents/data_api.html) for the API provides definitions of the different call parameters and examples of the various calls that can be made to the API. Additional information on the Air Quality System can be found in the [EPA FAQ](https://www.epa.gov/outdoor-air-quality-data/frequent-questions-about-airdata). Note that terms of use can be found [here](https://aqs.epa.gov/aqsweb/documents/data_api.html#terms). All data accessed through the API lies in the [public domain](https://edg.epa.gov/epa_data_license.html).

Specifically, we used the maximal daily average sensor data for monitoring stations in Deschutes County, all of which were <17 miles from Redmond, OR. These daily max values were then averaged for the fire season to get the annual estimate for 1983-2023. There was no data available before 1983. Finding nearby monitoring stations requires the Federal Information Processing Series (FIPS) of the desired city, county, and state. Information was gathered from [here](https://www.census.gov/library/reference/code-lists/ansi.html#cou). A detailed walkthrough of the data collection process can be found in [`./analysis_part1-AQI.ipynb`](./analysis_part1-AQI.ipynb). The final data can be found as [`./intermediate/final_annual_AQI_1983-2023.csv`](./intermediate/final_annual_AQI_1983-2023.csv).

## API Documentation
### EPA AQS API 
* [AQS API ](https://aqs.epa.gov/aqsweb/documents/data_api.html)
* [Documentation](https://aqs.epa.gov/aqsweb/documents/ramltohtml.html)
* [Additional AQS Info](https://www.epa.gov/aqs/aqs-manuals-and-guides)

## Code
The code for accessing and analyzing the data can be found in the following file:
* [`bias_analysis.ipynb`](./bias_analysis.ipynb): An interactive Jupyter Notebook providing a detailed walkthrough of the entire data acquisition, analysis, and storage process.

Note that some portions of the code were developed by Dr. David W. McDonald for use in DATA 512, a course in the UW MS Data Science degree program. This code was provided under the [Creative Commons](https://creativecommons.org) [CC-BY license](https://creativecommons.org/licenses/by/4.0/). The rest of the code lies under the standard [MIT license](./LICENSE).

## Output Files

### Data
The final merged data is stored as [`./output/wp_scored_city_articles_by_state.csv`](./output/wp_scored_city_articles_by_state.csv). It should be noted that this data doesn't contain any information for Nebraska or Connecticut since neither state has city article data. This data frame has 5 variables:
* `state`: The state of the city article.
* `regional_division`: The US Census Bureau subdivisions of US regions.
* `population`: The US Census Bureau estimate of state populations as of July 1, 2022.
* `article_title`: The title of the city article.
* `revision_id`: The revision ID of the city article.
* `article_quality`: The article quality as predicted by ORES.

## Intermediate Files
We store the raw JSON outputs of the API calls for page info and ORES scores just in case.
* [./intermediate/city_revids.json](./intermediate/city_revids.json): The dictionary of `pages` values from the page info JSON response keyed by an article's page ID. Ex.
```Python
{
    "104730": {                            # First city article
        "pageid": 104730,                  # Page ID of article
        "ns": 0,                           # Namespace (Main/Article)
        "title": "Abbeville, Alabama",     # Title of city's article
        "contentmodel": "wikitext",        # The content model (wikitext)
        "pagelanguage": "en",              # Language of the article (English)
        "pagelanguagehtmlcode": "en",      # Language of HTML code (English)
        "pagelanguagedir": "ltr",          # Language is left-to-write
        "touched": "2023-10-10T22:35:37Z", # When the data was accessed
        "lastrevid": 1171163550,           # The latest page revision ID
        "length": 24706                    # Length of the page
    },
    "104761": {                            # Second city article
        "pageid": 104761,
        "ns": 0,
        "title": "Adamsville, Alabama",
        "contentmodel": "wikitext",
        "pagelanguage": "en",
        "pagelanguagehtmlcode": "en",
        "pagelanguagedir": "ltr",
        "touched": "2023-10-10T22:35:37Z",
        "lastrevid": 1177621427,
        "length": 18040
    },
    ...
}
```
}
* [./intermediate/revid_scores.json](./intermediate/revid_scores.json): The dictionary of the ORES scores from the raw JSON response keyed by an article's revision ID. Ex.

```Python
{
    "1171163550": {                                # Revision ID  of article page
        "articlequality": {                        # The name of the model we use (predicting article quality)
            "score": {                             # We're interested in score
                "prediction": "C",                 # The ORES prediction of article quality
                "probability": {                   # The ORES model's probability of predicting each class
                    "B": 0.31042252456158204,        # B class
                    "C": 0.5979200965294227,         # C class
                    "FA": 0.025186220917133947,      # Feature Article
                    "GA": 0.04952133645299354,       # Good Article
                    "Start": 0.013573873336789355,   # Start class
                    "Stub": 0.0033759482020785892    # Stub class
                }
            }
        }
    },
    "1177621427": {                                # Revision ID of 2nd article page
        "articlequality": {
            "score": {
                "prediction": "C",
                "probability": {
                    "B": 0.198274200391586,
                    "C": 0.3770695177348356,
                    "FA": 0.019070364455845708,
                    "GA": 0.3514876684327692,
                    "Start": 0.05026148902798659,
                    "Stub": 0.003836759956977147
                }
            }
        }
    },
    ...
}
```

## Notes
### Smoke Estimation
We created an initial annual estimate of wildfire smoke in Redmond, OR to better understand the impact of wildfires on residents inside the city. Throughout the project, we will consider other socio-economic impacts as well. For this section, we only estimated the smoke seen by the city during each annual fire season from just the Wildfire data and recognized its limitations. Specifically, our final estimate is as follows:
$$s = \beta_0 + \beta_1 \frac{a}{d^2} + \beta_2 t,$$
where we let $s$ be the smoke experienced by the city due to a single fire, $a$ be the area, $d$ be the distance, and $t$ be the fire type. Additionally, we set $\beta$ as a tunable set of weights. Note that $\beta_0$ is the baseline amount of smoke present not attributed to wildfires, $\beta_1$ is the tunable fire-dispersal weight, and $\beta_2$ articulates the difference in baselines between different fire types. Ideally, if we know the levels of the known quantity we aim to model, we can tune the weights further. Currently, we use the following simple values for $\beta$: $\beta_0 = 0$, $\beta_1 = 1$, & $\beta_2 = 1$.

The above equation denotes the smoke effect for a single fire. To get an annual estimate, we sum the total smoke effects of all fires in a given year and divide by the number of days in the fire season. Summing the values approximates the total smoke intake by the city throughout the fire season. 
Dividing this by the number of days (184) in the fire season (May 1st through October 31st) could in turn approximate the average daily smoke quality for the city during the fire season. Note that this simplified average assumes that each fire's duration and inception were equal which is a fundamentally erroneous assumption but might make for a decent estimate.

#### Rationale
Typically, smoke is dependent on wind patterns over several days, the intensity of the fire, its duration, and the distance from the city. However, for the sake of this assignment, we only have access to the fire area & distance. Additionally, we can distinguish between the type of fire as a proxy for fire intensity: prescribed fires and true wildfires. Prescribed burns are conducted on days where weather conditions are optimal as a way to mitigate safety risks and the spread of smoke. [\[1\]](https://extension.okstate.edu/fact-sheets/the-best-time-of-year-to-conduct-prescribed-burns.html)  As such, prescribed fires can be assumed to contribute less to the smoke drifting over nearby cities than wildfires. We also know that larger fires near the city will contribute more to smoke quantity over a city than small fires further away. However, how much do we estimate each factor, area & distance, to contribute to the overall quantity? 

Smoke is generated from incomplete combustion, denoted by the formula, $\text{Fuel} + O_2 \rightarrow CO_2 + H_2O + \text{byproducts}$. [\[2\]](https://www.sciencelearn.org.nz/resources/748-what-is-smoke)  This tells us that smoke is **linearly proportional** to the amount of fuel burned, which in turn is linearly proportional to the area burned. Meanwhile, the intensity of energy, force, or flux evenly radiated from a source follows an inverse-square law with distance as commonly observed with light. [\[3\]](https://en.wikipedia.org/wiki/Inverse-square_law)  We can model smoke as flux originating from the fire and evenly radiating outward from the burned area. So, the smoke estimate of the city can be an **inverse square of the distance** between the city and the fire. Putting the above assumptions together yields our initial estimate for smoke from a single fire:
$$\text{smoke} \propto \frac{\text{area}}{\text{distance}^2}$$

However, we know that the type of fire drastically changes its dispersal over a city. We can model this as a varying baseline for each fire type:
$$\text{smoke} \propto \frac{\text{area}}{\text{distance}^2} + \text{fire type}$$

*Note that while area and distance might have different imperial units, we can ignore the conversion as this is something that can be tuned with* $\beta$.

### AQI Aggregation
A summary AQI index data was provided for a series of pollutants (3 particulate & 2 gaseous).
* Particulate: Acceptable PM2.5 AQI & Speciation Mass, PM2.5 - Local Conditions, & PM10 Total 0-10um STP
* Gaseous: Carbon Monoxide & Ozone
  *  Nitrous Oxide and Sulfur dioxide sensors weren't available.
Wildfire smoke is mainly composed of fine (PM 2.5) particles (>90% by mass) but also contains some percentage of coarse particles (PM10 particles) and a small percentage of gaseous pollutants as well. [Source 1](https://www.scientificamerican.com/article/wildfire-smoke-reacts-with-city-pollution-creating-new-toxic-air-hazard/#:~:text=Scientists%20have%20long%20known%20that,it%20blocks%20harmful%20ultraviolet%20rays.), [Source 2](https://www.scientificamerican.com/article/wildfire-smoke-reacts-with-city-pollution-creating-new-toxic-air-hazard/#:~:text=Scientists%20have%20long%20known%20that,it%20blocks%20harmful%20ultraviolet%20rays.)

Since we don't have a good understanding of the exact proportion of each pollutant's contribution to smoke & that the overall reported AQI is the [maximum value of the AQI for each subcategory](https://www.airnow.gov/sites/default/files/2020-05/aqi-technical-assistance-document-sept2018.pdf), we'll make our estimate to be the highest AQI for any given day from any of the five stations near the city.

### Python & Jupyter Set-Up
This work assumes that users have a working Jupyter Notebook & Python 3 setup. Instructions on installing them can be found [here](https://docs.jupyter.org/en/latest/install/notebook-classic.html). It should be noted that Python modules required for this work comprise some standard modules that are installed with Python and others that are installed through the [Anaconda](https://docs.jupyter.org/en/latest/install/notebook-classic.html) distribution. 

If modules are not found, they can be readily installed with the following terminal commands:

```bash
    pip install <module name>
```
or 
```bash
   conda install <module name>
```

### EPA AQS API KEY
Please note that an account tied to an email is required to use the API. Steps for setting up the API Key were detailed by Dr. McDonald:
1. Create an email address & request an API key using the EPA endpoint & function defined below. 
```Python
import json
def request_signup(email_address = None,
                   endpoint_url = API_REQUEST_URL, 
                   endpoint_action = API_ACTION_SIGNUP, 
                   request_template = AQS_REQUEST_TEMPLATE,
                   headers = None):
    """
    Function request access using an email address. 
    The parameters are standardized so that this function definition matches all of the others. 
    However, the easiest way to call this is to simply call this function with your preferred email address.
    Parameters
        email_address (str): The email address to use for the sign-up request.
        endpoint_url (str): The base URL of the API endpoint.
        endpoint_action (str): The specific action or endpoint for the sign-up request.
        request_template (dict): A dictionary containing request parameters and values.
        headers (dict): Optional headers to include in the request.
        
    Returns:
    - dict or None: A JSON response containing the sign-up request process
        Returns None if there is an exception during the request.

    Raises:
        Exception: If any required parameters are missing.
    """
    # Make sure we have a string - if you don't have access to this email address, things might go badly for you
    if email_address:
        request_template['email'] = email_address        
    if not request_template['email']: 
        raise Exception("Must supply an email address to call 'request_signup()'")
    
    # Compose the signup url - create a request URL by combining the endpoint_url with the parameters for the request
    request_url = endpoint_url+endpoint_action.format(**request_template)
        
    # make the request
    try:
        # Wait first, to make sure we don't exceed a rate limit in the situation where an exception occurs
        # During the request processing - throttling is always a good practice with a free data source
        if API_THROTTLE_WAIT > 0.0:
            time.sleep(API_THROTTLE_WAIT)
        response = requests.get(request_url, headers=headers)
        json_response = response.json()
    except Exception as e:
        print(e)
        json_response = None
    return json_response
response = request_signup("ymanne@uw.edu")
```
2. Validate email using the link EPA sends. Once the API Key token is created, store your Wikimedia username and access token as `USERNAME` and `APIKEY` string variables in a file called `my_secrets.py`. This file & the two variables are imported into the code but should never be published publically. 
