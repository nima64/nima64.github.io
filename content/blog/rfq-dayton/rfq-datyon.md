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
In one week, Mario and I built a mvp quoting software for Dayton Financial. We converted their laborious process from entering seller data in a buyer sheet into a streamlined full stack web app. It saved the company thousands of dollars in monthly operation costs. 

The vision for the project was to have a master sheet with products that sellers could enter their quotes and they would be grouped and displayed. This setup of private sellers, a master sheet where any changes would reflect seller sheets and providing real time updates required real engineering and man hours to build the mvp in one week.

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
Arguably the hardest part of this project was modeling buyers and sellers. A seller's copies are contained inside buyer/template sheets. Because buyer sheets template updates are unidirectional, any update in the template must be applied to all seller sheets but not vice versa.
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
We made the assumption of at most 100 users at any given time, so we had to figure out a way to limit our API requests to stay under Firebase's 20k requests per hour limit. If we sent updates every second, we would easily hit this limit, so we decided to implement a buffer that would hold the changes. We check this buffer every 3 seconds for changes and make a single API request if any exist.

{% image "./seller-edit.png" ,"seller-edit"%}

This approach had one big issue: race conditions. Luckily my background from Google Summer of Code helped me navigate this bug. The problem was that user changes could be added to the buffer during an API request, but then get cleared immediately after when we reset the buffer. This meant those changes would never be processed.
The solution was to implement a two-buffer system: a processed buffer that gets cleared immediately before each API call and a queque holds any changes that occur during the request, preventing data loss.

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
### In house Tanstack grid vs commercial grid

Towards the end of the project, we had to implement row grouping and column sorting. Mui grid and Ag-grid were upwards of 500-1k annually, and they were all built on top of tanstack. So naturally being the crafty engineers we were, we build our own grid on top of Tan Stack this way if we ever need to make changes to the grid we knew exactly how it worked with the added expense of technical debt hahahaha.