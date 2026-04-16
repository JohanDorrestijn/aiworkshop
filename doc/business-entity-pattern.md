# ABL Business Entity Architecture Pattern

## Overview

The Business Entity pattern provides a standardized, maintainable approach to data access in OpenEdge ABL applications. It separates UI logic from database operations through a layered architecture that promotes reusability, testability, and consistency across the application.

## Architecture Layers

### 1. UI Layer (Windows/Forms)
- **Responsibility**: User interaction and presentation
- **Access**: Never directly accesses database tables
- **Communication**: Calls Business Entity methods with datasets

### 2. Business Entity Layer
- **Responsibility**: Data access, business rules, validation
- **Inheritance**: Extends `OpenEdge.BusinessLogic.BusinessEntity`
- **Management**: Instantiated through EntityFactory (singleton pattern)

### 3. Database Layer
- **Responsibility**: Persistent storage
- **Access**: Only through data-sources attached to business entities

## Key Components

### EntityFactory (Singleton Pattern)

**Purpose**: Centralized management of business entity lifecycle

**Pattern**:
```abl
CLASS business.EntityFactory:
    /* Singleton instance */
    VAR PRIVATE STATIC EntityFactory objInstance.
    
    /* Entity instances */
    VAR PRIVATE CustomerEntity objCustomerEntityInstance.
    
    /* Private constructor prevents direct instantiation */
    CONSTRUCTOR PRIVATE EntityFactory():
    END CONSTRUCTOR.
    
    /* Public factory method */
    METHOD PUBLIC STATIC EntityFactory GetInstance():
        IF objInstance = ? THEN
            objInstance = NEW EntityFactory().
        RETURN objInstance.
    END METHOD.
    
    /* Entity getters with lazy initialization */
    METHOD PUBLIC CustomerEntity GetCustomerEntity():
        IF objCustomerEntityInstance = ? THEN
            objCustomerEntityInstance = NEW CustomerEntity().
        RETURN objCustomerEntityInstance.
    END METHOD.
END CLASS.
```

**Benefits**:
- Single source of truth for entity instances
- Prevents memory waste from duplicate objects
- Enables centralized cleanup and testing

### Dataset Definition (.i Include Files)

**Purpose**: Defines temp-tables and datasets for data transfer

**Pattern**:
```abl
/* Define temp-table with BEFORE-TABLE for change tracking */
DEFINE TEMP-TABLE ttCustomer BEFORE-TABLE bttCustomer
    FIELD CustNum AS INTEGER INITIAL "0" LABEL "Cust Num"
    FIELD Name AS CHARACTER LABEL "Name"
    FIELD Address AS CHARACTER LABEL "Address"
    /* ... additional fields ... */
    INDEX CustNum IS PRIMARY UNIQUE CustNum ASCENDING.

/* Define dataset containing the temp-table */
DEFINE DATASET dsCustomer FOR ttCustomer.
```

**Key Points**:
- `BEFORE-TABLE` enables change tracking for updates
- Temp-table fields match database table structure
- Primary index mirrors database primary key
- Shared via include file for consistency

### Business Entity Class

**Purpose**: Encapsulates all data operations for a specific entity

**Pattern**:
```abl
CLASS business.CustomerEntity INHERITS BusinessEntity USE-WIDGET-POOL:
    
    /* Include dataset definition */
    {business/CustomerDataset.i}
    
    /* Define data sources - one per database table */
    DEFINE DATA-SOURCE srcCustomer FOR Customer.
    
    /* Constructor */
    CONSTRUCTOR PUBLIC CustomerEntity():
        SUPER(DATASET dsCustomer:HANDLE).
        
        VAR HANDLE[1] hDataSourceArray = DATA-SOURCE srcCustomer:HANDLE.
        VAR CHARACTER[1] cSkipListArray = [""].
        
        THIS-OBJECT:ProDataSource = hDataSourceArray.
        THIS-OBJECT:SkipList = cSkipListArray.
    END CONSTRUCTOR.
    
    /* CRUD Operations */
    METHOD PUBLIC LOGICAL GetCustomerByName(INPUT ipcCustomerName AS CHARACTER, 
                                            OUTPUT DATASET dsCustomer):
        VAR CHARACTER cFilter = "WHERE Customer.Name = '" + ipcCustomerName + "'".
        THIS-OBJECT:ReadData(cFilter).
        RETURN CAN-FIND(FIRST ttCustomer).
    END METHOD.
    
    METHOD PUBLIC VOID CreateCustomer(INPUT-OUTPUT DATASET dsCustomer):
        THIS-OBJECT:CreateData(DATASET dsCustomer BY-REFERENCE).
    END METHOD.
    
    METHOD PUBLIC VOID UpdateCustomer(INPUT-OUTPUT DATASET dsCustomer):
        THIS-OBJECT:UpdateData(DATASET dsCustomer BY-REFERENCE).
    END METHOD.
    
    METHOD PUBLIC VOID DeleteCustomer(INPUT-OUTPUT DATASET dsCustomer):
        THIS-OBJECT:DeleteData(DATASET dsCustomer BY-REFERENCE).
    END METHOD.
    
END CLASS.
```

## UI Integration Pattern

**UI Window Structure**:
```abl
/* Include business entity classes */
USING business.CustomerEntity FROM PROPATH.
USING business.EntityFactory FROM PROPATH.

/* Include dataset definitions */
{business/CustomerDataset.i}

/* UI Event Handler */
ON CHOOSE OF GetCustomer IN FRAME DEFAULT-FRAME DO:
  VAR INTEGER iCustomerNumber = INTEGER(CustomerNumber:screen-value).
  VAR EntityFactory objFactory = EntityFactory:GetInstance().
  VAR CustomerEntity objCustomerEntity = objFactory:GetCustomerEntity().
  VAR LOGICAL lCustomerFound = objCustomerEntity:GetCustomerByNumber(iCustomerNumber, 
                                                                  OUTPUT DATASET dsCustomer).
  
  IF lCustomerFound THEN DO:
    FIND FIRST ttCustomer.
    IF AVAILABLE ttCustomer THEN DO:
      CustomerName = ttCustomer.Name.
      DISPLAY CustomerName WITH FRAME {&frame-name}.
    END.
  END.
  ELSE 
    MESSAGE "Customer not found" VIEW-AS ALERT-BOX.
END.
```

## Best Practices

### 1. Data Access
- Always use named buffers for database access
- Never access database tables directly from UI
- Use temp-tables for all data transfer between layers
- Enable change tracking with BEFORE-TABLE for updates

### 2. Error Handling
- Use BLOCK-LEVEL ON ERROR UNDO, THROW in classes
- Implement validation methods in business entities
- Return logical values for success/failure operations
- Provide meaningful error messages to UI layer

### 3. Performance
- Use singleton pattern for entity instances
- Pass datasets by reference to avoid copying
- Use appropriate data-source configurations
- Implement proper indexing on temp-tables

### 4. Maintainability
- Group related functionality in business entities
- Use consistent naming conventions
- Document all public methods and parameters
- Follow the established patterns for new entities

## Benefits

1. **Separation of Concerns**: UI focuses on presentation, business entities handle data
2. **Reusability**: Business entities can be used by multiple UI components
3. **Testability**: Business logic can be tested independently of UI
4. **Maintainability**: Changes to data access logic are centralized
5. **Consistency**: Standardized approach across all entities
6. **Performance**: Efficient data access through optimized data-sources

## Implementation Checklist

- [ ] Create EntityFactory singleton class
- [ ] Define dataset include files with temp-tables
- [ ] Implement business entity classes inheriting from BusinessEntity
- [ ] Configure data-sources for each database table
- [ ] Implement CRUD methods in business entities
- [ ] Update UI windows to use EntityFactory and business entities
- [ ] Add validation methods in business entities
- [ ] Test all data operations through UI layer
- [ ] Verify error handling and user feedback