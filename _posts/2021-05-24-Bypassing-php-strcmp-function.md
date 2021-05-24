---
layout: post
title:  "Bypassing php strcmp function"
date:   2021-05-24 12:11:51 +0530
categories: CTF php
---
<style type="text/css">
  img {
    padding: 5px;
    display: block;
  }
</style>
In this post, I will explain how we can bypass the strcmp function in php. This challenge was a part of the natas wargames level 24.

<h4><u>Challenge</u></h4>
1. We are given a web page where we are expected to enter the password for the next level (level 25).
2. Clicking on view source code reveals the following code. The html part is omitted for brevity.
```php
<?php
    if(array_key_exists("passwd",$_REQUEST)){
        if(!strcmp($_REQUEST["passwd"],"<censored>")){
            echo "<br>The credentials for the next level are:<br>";
            echo "<pre>Username: natas25 Password: <censored></pre>";
        }
        else{
            echo "<br>Wrong!<br>";
        }
    }
    // morla / 10111
?>  
```

<h4><u>Solution</u></h4>
1. In the above code, the password entered by the user is passed to the strcmp function. The strcmp function then checks the input against the hardcoded password.  
2. According to the [php documentation][strcmp-documentation], strcmp either returns 0, positive number or negative number. Now given the fact that php considers both postive numbers and negative numbers to be true, we either need an exact match or some other flaw in strcmp.
3. The flaw is that strcmp behaves unpredictably when a string is compared to any other data type. So to solve the challenge, we have to change the 'passwd' parameter in the url to 'passwd[]'. This reveals the password for next level.
<img src="{{ site.baseurl }}/assets/images/2_1_solution.png" alt="natas_24_soln">

[strcmp-documentation]: https://www.php.net/manual/en/function.strcmp.php

