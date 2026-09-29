Yes. Levels.fyi supports structured query parameters directly in its search bar (particularly on the main salaries/data explorer page) using key-value filters and search operators.

  

Here are the key syntax rules and common filters supported:

  

### Common Filter Keys

You can filter by specific fields using `key:value` syntax:

  

- **Company:** `company:Google` or `company:"Meta Platforms"` (use quotes for multi-word names)
    
      
    
- **Title / Role:** `title:"Software Engineer"` or `title:Product\ Manager`
    
      
    
- **Level:** `level:L5` or `level:Senior`
    
      
    
- **Location / City / Region:** `city:"San Francisco"` or `country:"United States"` or `region:"Greater Seattle Area"`
    
      
    
- **Years of Experience (YOE):**
    
      
    - Exact: `yoe:5`
        
          
        
    - Range/Inequality: `yoe:>5`, `yoe:<3`, `yoe:>=4`
        
          
        
- **Years at Company:** `yac:>2`
    
      
    
- **Compensation / Total Comp:**
    
      
    - `totalcompensation:>250k` or `totalcompensation:>300000`
        
          
        
    - Base salary: `basesalary:>180k`
        
          
        
    - Stock: `stockgrantvalue:>50k`
        
          
        
    - Bonus: `bonus:>20k`
        
          
        
- **Date / Recency:** `date:>2023-01-01`
    
      
    
- **Tags / Focus:** `tag:Android`, `tag:Distributed\ Systems`
    
      
    

### Operators & Boolean Logic

- **AND:** Queries are treated as `AND` by default when space-separated:
    
      
    - `company:Google title:"Software Engineer" yoe:>5`
        
          
        
- **OR:** Supported between values or terms (often capitalized):
    
      
    - `company:(Google OR Meta)`
        
          
        
- **Negation (`-`):** Exclude terms or companies:
    
      
    - `title:"Software Engineer" -company:Amazon`
        
          
        
- **Quotes (`"..."`):** Exact phrase match for names with spaces:
    
      
    - `company:"Jane Street" title:"Quantitative Trader"`
        
          
        

### Example Queries

- **Senior engineers in NYC making over $300k:**
    
    `title:"Software Engineer" city:"New York" totalcompensation:>300k yoe:>5`
    
      
    
- **Comparing L4/L5 at specific companies:**
    
    `company:(Google OR Meta) (level:L4 OR level:L5 OR level:E4 OR level:E5)`
    
      
    
- **Remote roles excluding Amazon:**
    
    `tag:Remote -company:Amazon title:"Product Manager"`
    
      
    

If you have a specific query or target role/dataset you are trying to construct, let me know!

title: "Network Software Engineer"  level: senior city:"San Francisco"