# The dot symbol





The dot `.` appears in different places in the R ecosystem: e.g. purrr, magittr's `%>%`. I will explore and explain what happens if you mix these usages, or nest them, how the dot symbol is special and how it is not.

## Basic usage

### %>% of magrittr

You should use the dot if the parameter you pipe forward is not the first parameter of your next function or if you use pipe with data.table and use `[]` function.


```r
lipsum::lipsums[1] %>% 
    stringi::stri_trans_totitle() %>% 
    gsub(' ', '-', .)
```

```
## [1] "Lorem-Ipsum-Dolor-Sit-Amet,-Consectetur-Adipiscing-Elit."
```

You can also refer to the parameter with `.` if it is the first, but then you do not have to:


```r
lipsum::lipsums[1] %>%
    stringi::stri_trans_totitle(.) %>% 
    stringr::str_replace_all(., ' ', '-')
```

```
## [1] "Lorem-Ipsum-Dolor-Sit-Amet,-Consectetur-Adipiscing-Elit."
```

### Map of purrr


```r
calendar_consts <- list(
    'num_hours_in_day' = 24,
    'num_days_in_week' = 7
)
map_chr(names(calendar_consts), ~ stringr::str_c(., ': ', calendar_consts[[.]]))
```

```
## [1] "num_hours_in_day: 24" "num_days_in_week: 7"
```

For compact anonymous functions the formula notation with `.` is a shorthand for the more verbose following known from the base R `apply` family.


```r
map_chr(
    names(calendar_consts),
    function(name) {stringr::str_c(name, ': ', calendar_consts[[name]])}
)
```

```
## [1] "num_hours_in_day: 24" "num_days_in_week: 7"
```

Of course defining your function outside the call to map is always possible and preferable for more complex functions.

## Nested usage

### Map within map


```r
consts <- list(
    'calendar' = calendar_consts,
    'geo' = list(
        'num_continents' = 7,
        'num_states_in_US' = 50
    )
)
map(
    consts,
    ~{
        sub_list <- .
        map_chr(
        names(sub_list),
        ~ stringr::str_c(., ': ', sub_list[[.]])
    )}
)
```

```
## $calendar
## [1] "num_hours_in_day: 24" "num_days_in_week: 7" 
## 
## $geo
## [1] "num_continents: 7"    "num_states_in_US: 50"
```

`.` is actually not that different from ordinary variable names which means that scoping rules apply as usual. In the inner if you refer to `.` it means the current element in the inner map. You can either save the outer current element to a variable or define the desired environment for your variable name.


```r
map(
    consts,
    ~ map_chr(
        names(.),
        ~ stringr::str_c(., ': ', parent.env(environment())$'.'[[.]])
    )
)
```

```
## $calendar
## [1] "num_hours_in_day: 24" "num_days_in_week: 7" 
## 
## $geo
## [1] "num_continents: 7"    "num_states_in_US: 50"
```

While the above is possible I do not recommend it as it is difficult to read.
You could instead do the following:


```r
pasteConstNamesAndValues <- function(const_list) {
    map_chr(
        names(const_list),
        ~ stringr::str_c(., ': ', const_list[[.]])
    )  
}
map(consts, pasteConstNamesAndValues)
```

```
## $calendar
## [1] "num_hours_in_day: 24" "num_days_in_week: 7" 
## 
## $geo
## [1] "num_continents: 7"    "num_states_in_US: 50"
```

### Pipe within pipe

I could not came up with a realistic usage for this as pipe is exactly for avoiding nesting...

## Mixed usage

### Pipe in map


```r
map_chr(
    names(calendar_consts),
    ~ {
        stringr::str_replace_all(., '_', ' ') %>% 
            stringr::str_c(': ', calendar_consts[[.]]) 
    }
)
```

```
## [1] "num hours in day: " "num days in week: "
```

The above does not work as intended as `.` in `calendar_consts[[.]]` refers to the variable forwarded by ` %>% `, in this case the already transformed variable name. Luckily we can refer to the current element in map with `.x` as well.


```r
map_chr(
    names(calendar_consts),
    ~ {
        stringr::str_replace_all(., '_', ' ') %>% 
            stringr::str_c(': ', calendar_consts[[.x]]) 
    }
)
```

```
## [1] "num hours in day: 24" "num days in week: 7"
```

### Map in pipe


```r
names(calendar_consts) %>% 
    map(., ~ stringr::str_c(., ': ', calendar_consts[[.]]))
```

```
## [[1]]
## [1] "num_hours_in_day: 24"
## 
## [[2]]
## [1] "num_days_in_week: 7"
```

What if I want to use the forward-piped object inside the body of map, not just mapping over it? Then the two `.` symbols will really conflict. We can avoid this by extracting the map into a named function:


```r
pasteWithSeparators <- function(const_list, separators) {
    map(
        names(const_list),
        ~ stringr::str_c(., separators, const_list[[.]])
    ) 
}
```


```r
c(': ', ' -- ') %>% 
    pasteWithSeparators(calendar_consts, .)
```

```
## [[1]]
## [1] "num_hours_in_day: 24"   "num_hours_in_day -- 24"
## 
## [[2]]
## [1] "num_days_in_week: 7"   "num_days_in_week -- 7"
```

But it won't work if we inline the function:


```r
c(': ', ' -- ') %>% 
    map(
        names(calendar_consts),
        ~ stringr::str_c(., ., calendar_consts[[.]])
    ) 
```

```
## [[1]]
## NULL
## 
## [[2]]
## NULL
```

Apparently by using the pipe you have to formally pass `.` as a variable to your function otherwise it will be passed as first, not our intention here. The solution is to adding curly braces:


```r
c(': ', ' -- ') %>% 
    {map(
        names(calendar_consts),
        ~ stringr::str_c(., c(': ', ' -- '), calendar_consts[[.]])
    )} 
```

```
## [[1]]
## [1] "num_hours_in_day: 24"   "num_hours_in_day -- 24"
## 
## [[2]]
## [1] "num_days_in_week: 7"   "num_days_in_week -- 7"
```

We still have to figure out how to pass the separators to `map` as the `.` will refer to the current element in map. Fortunately we can refer to the parent environment here as well. But this is very difficult to read, so either do not use the pipe in such cases or predefine your functions. 


```r
c(': ', ' -- ') %>% 
    {map(
        names(calendar_consts),
        ~ stringr::str_c(., parent.env(environment())$'.', calendar_consts[[.]])
    )} 
```

```
## [[1]]
## [1] "num_hours_in_day: 24"   "num_hours_in_day -- 24"
## 
## [[2]]
## [1] "num_days_in_week: 7"   "num_days_in_week -- 7"
```

## Conclusions

`.` behaves like a normal variable name but comes handy when communicating clear patterns in a compact way.
