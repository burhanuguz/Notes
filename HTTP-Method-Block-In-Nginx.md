# HTTP Method Block in Nginx
- For secure environments these blocking should be made. 
```nginx
server {
  
  ..........
  ..........
  
  location example_location1 .... {
    limit_except GET POST OPTIONS {
      deny  all;
    }
   }

  
   location example_location2 .... {
     limit_except GET POST OPTIONS {
       deny  all;
     }
   }

  ..........
  ..........
    
}
```
- In this example **GET**, **POST** and **OPTIONS** methods are blocked from two locations. There can be done another limits or blockings location-wise.
