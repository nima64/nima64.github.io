---
title: Dayton Sheets
description: Moving google Sheets Work Flow to custom RFQ Web App 
date: 2025-06-30
tags:
  - RFQ, Custom Qouting Software, MERN, firebase, typescript, Nextjs
---

{% image "./offers.png" ,"offers"%}
*(buyer view)*
## Building Dayton Sheets
In one week, Mario and I built a mvp quoting software for Dayton Financial. We converted their labourous process from entering seller data in a buyer sheet into a streamlined full stack webapp. It saved the company thousands of dollars in monthly operation costs. 

The vision for the project was to have master sheet with products that would sellers could enter their qoutes and they would be grouped and dispalyed. This setup of private sellers, a master sheet where any changes would reflect seller sheets and providing real time updates required real engineering and man hours to build the mvp in one week.

We decided the on the following stack: **Firebase, Tanstack, Nextjs, Typescript, Reactjs**

### Some of the challenges and constraints we faced 
- Modeling data seller & master sheets
- Data synchronizaiton between master & seller
- responsive spreadsheet component(under 100ms render for input)
- can columns change?
- handle up to 100 sellers at anytime
    - at most 20,000 api firebase requests in an hour(Free tier limit)  

How we solved these problems will be discussed in this blog

### Modeling Sellers and Buyers

{% image "./ERD.png" ,"ERD"%}
Arguably the hardest part of this project was modeling buyers and sellers. A seller copies are contained inside buyer/template sheets. Because buyer sheets are templates updates are unidirectional, any update in the template must be applied to all seller sheets but not vise versa.
We decided to make the changes on the seller sheets only store the rows that were changed. The buyer display will take those changes and group them by their row id.


``` javascript
 MasterSheet {
  templateId:  string;       // PK
  buyerId:     string;       // FK → Buyer.userId
  title:       string;
  columns:     string[];
  rows:        Row[];
  sellers:     string[];     // UserIds
  sellerCopies?: Record<string, Row[]>; // optional per-seller view
}

 SellerCopy {
  templateId: string;  // PK (composite)
  sellerId:   string;  // PK
  rows:       Row[];
}
/// rest in on github....
```
<br />

### Handling up to 100 users and real time updates
We made the assumption of at most 100 users at any given time, we had to figure out a way to limit our api requests to prevent the 20k requests per hour limit. If we sent updates on the sheet ever second our api would easily hit the limit, so we decided to make a buffer that would hold the changes. We would check this buffer every 3 seconds for changes and if changes exist make api request.

{% image "./seller-edit.png" ,"seller-edit"%}

This had one big issue race conditions... Lucky my background from Google summer of code helpled me navigate this bug. The problem was that changes could occur were added to the buffer could happen *during* API requests, then they would be cleared immediatly after to reset the buffer. If they were added during the api requests, they were would never get to be able to be processed. So the solution was make an pre processing queue that would be cleared immediatley before the api call and contain all changes during the api call.



``` typescript
const queuedCells = new Map<string, any>();
const processedCells = new Map<string, any>(); // Pending API updates

setInterval(() => {
    if (queuedCells.size === 0) return;

    // Move user changes from queued to processed
    queuedCells.forEach((update, key) => {
      processedCells.set(key, update); // Latest value wins
    });

    queuedCells.clear();

    //async can trigger can the person is making changes to row
    makeApiCall("/api/sheet/batch-update?")
      .then((data) => {
        processedCells.clear();
      })
      .catch((err) => {
        // dump processed cells back into queue  
        processedCells.forEach((update, key) => {
          queuedCells.set(key, update); 
        });
      })
}, 3000);
```
### In house Tanstack grid vs comercial grid

Towards the end of the project, we had to implement row grouping and column sorting. Mui grid and Ag-grid were upwards of 500-1k annually, and they were all built on top of tanstack. So naturally being the crafty engineers we were we bulid our own grid on top of Tanstack this way if we ever need to make changes to the grid we knew exactly how it worked with the added expense of technical debt hahahaha.
