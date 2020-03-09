---
title: "Calculating sunrise"
description: A transliteration of a sunrise/sunset time algoritm into C++
date: '2020-03-20'
---

# How is sunrise/sunset time calculated?

A Google search turns up
http://www.edwilliams.org/sunrise_sunset_algorithm.htm, which looks to
be authoritative:

```
Source:
	Almanac for Computers, 1990
	published by Nautical Almanac Office
	United States Naval Observatory
	Washington, DC 20392
```

## Inputs

The day, month and year are used to find the day number (between 1 and
366). Latitude and longitude determine our location, and zenith
clarifies what we actually mean by _sunset_ and _sunrise_.

```
	day, month, year:      date of sunrise/sunset
	latitude, longitude:   location for sunrise/sunset
	zenith:                Sun's zenith for sunrise/sunset
	  offical      = 90 degrees 50'
	  civil        = 96 degrees
	  nautical     = 102 degrees
	  astronomical = 108 degrees
	
        NOTE: longitude is positive for East and negative for West
        NOTE: the algorithm assumes the use of a calculator with the
        trig functions in "degree" (rather than "radian") mode. Most
        programming languages assume radian arguments, requiring back
        and forth convertions. The factor is 180/pi. So, for instance,
        the equation RA = atan(0.91764 * tan(L)) would be coded as RA
        = (180/pi)*atan(0.91764 * tan((pi/180)*L)) to give a degree
        answer with a degree input for L.
```

## 1. first calculate the day of the year

```
	N1 = floor(275 * month / 9)
	N2 = floor((month + 9) / 12)
	N3 = (1 + floor((year - 4 * floor(year / 4) + 2) / 3))
	N = N1 - (N2 * N3) + day - 30
```

`floor` is truncating division: you throw away the remainder. C++ has
two forms of division (confusingly both are written using the same
symbol, `/`). Dividing an integer by another integer produces an
integer (truncated). Dividing a floating point number by anything
produces a floating point. So, `3 / 2` is 1 but `3.0 / 2` is 0.5.

### Defining dayOfYear as a function

```cpp
int dayOfYear(int day, int month, int year) {
  // exercise: transliterate the above algorithm into C++
  return N;
}
```

### Testing

It's not at all clear that this algorithm does what it says, so we
should test it. We can do that by printing out the day number for a
few interesting cases, such as month boundaries and leap years. March
1 is a good test case, as it is the first date affected by a leap
year. We expect $31 + 28 + 1$ in a normal year and $31 + 29 + 1$ in a
leap year.

```cpp
#include <iostream>

int main() {
  std::cout << "dayOfYear(1, 3, 2020) = " << dayOfYear(1, 3, 2020)
            << "dayOfYear(1, 3, 2020) = " << dayOfYear(1, 3, 2019)
			<< std::endl;
  return 0;
}
```

*Question*: Does the algorithm apply leap years on centuries correctly?
2000 was a leap century, but 1900 was not (every 400 years is a leap
century; other centuries are not).

## 2. convert the longitude to hour value and calculate an approximate time

```
	lngHour = longitude / 15
	
	if rising time is desired:
	  t = N + ((6 - lngHour) / 24)
	if setting time is desired:
	  t = N + ((18 - lngHour) / 24)
```

## 3. calculate the Sun's mean anomaly

```
	M = (0.9856 * t) - 3.289
```

## 4. calculate the Sun's true longitude

```
	L = M + (1.916 * sin(M)) + (0.020 * sin(2 * M)) + 282.634
	NOTE: L potentially needs to be adjusted into the range [0,360) by adding/subtracting 360
```

We need to converting degrees to radians (and back). The algorithm
assumes we are using degrees, while the C++ math library assumes
radians. We can wrap the library functions, using an Uppercase first
letter to distinguish them, to isolate the conversion.

```cpp
#define _USE_MATH_DEFINES
#include <cmath>

double Sin(double degrees) { return sin(degrees * M_PI_2); }
// etc. for the remaining trigonometric functions
```

## 5a. calculate the Sun's right ascension

```
	RA = atan(0.91764 * tan(L))
	NOTE: RA potentially needs to be adjusted into the range [0,360) by adding/subtracting 360
```

## 5b. right ascension value needs to be in the same quadrant as L

```
	Lquadrant  = (floor( L/90)) * 90
	RAquadrant = (floor(RA/90)) * 90
	RA = RA + (Lquadrant - RAquadrant)
```

## 5c. right ascension value needs to be converted into hours

```
	RA = RA / 15
```

## 6. calculate the Sun's declination

```
	sinDec = 0.39782 * sin(L)
	cosDec = cos(asin(sinDec))
```

## 7a. calculate the Sun's local hour angle

```
	cosH = (cos(zenith) - (sinDec * sin(latitude))) / (cosDec * cos(latitude))
	
	if (cosH >  1) 
	  the sun never rises on this location (on the specified date)
	if (cosH < -1)
	  the sun never sets on this location (on the specified date)
```

## 7b. finish calculating H and convert into hours
	
```
	if if rising time is desired:
	  H = 360 - acos(cosH)
	if setting time is desired:
	  H = acos(cosH)
	
	H = H / 15
```

## 8. calculate local mean time of rising/setting

```
	T = H + RA - (0.06571 * t) - 6.622
```

## 9. adjust back to UTC

```
	UT = T - lngHour
	NOTE: UT potentially needs to be adjusted into the range [0,24) by adding/subtracting 24
```

## 10. convert UT value to local time zone of latitude/longitude

```
	localT = UT + localOffset
```

# Plotting sunrise

Calculate the sunrise time for every latitude for every day of the year.
```cpp
for (auto N = 0; N < 366l N++) {
  for (auto latitude = 0; latitude < 90; latitude++)
    cout << sunrise(N, latitude, 0) << ",";
  cout << "-1\n";
}
```

## Using R

```R
install.packages("plotly")
library(plotly)
d <- read.csv("sunrise-times.csv", header=FALSE, na.strings = "-1")
plot_ly(z = ~as.matrix(d)) %>% add_surface()
```

## We can do the computation in parallel

```bash
g++ -fopenmp sunrise.cpp -o sunrise
```

We can't output the results in parallel, but we can compute them and
save the values into an array:

```cpp
int main() {
  double results[365][90];
#pragma omp parallel for
  for (auto day = 0; day < 365; day++)
    for (auto latitude = 0; latitude < 90; latitude++)
      results[day][latitude] = sunrise(day, latitude, 0);
```

You can read more about OpenMP at, e.g. https://bisqwit.iki.fi/story/howto/openmp/.
