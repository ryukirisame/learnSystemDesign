# Optimistic Locking
- This type of locking assumes that conflicts are rare. So, each transaction can read freely. But we must verify before writing.
- This is not actually a lock as in traditional sense.
- The idea is this: Each row will include a token number (version, timestamp etc). Whenever a transaction reads a row, it will also read this version number. All is fine till now.
  - But when the same transaction tries to update the row, it will have to verify if the token number it read matches the token number stored in the table or not.
  - If the token number matches that means the row was not updated since the transaction read the row. So, the transaction can proceed with updating the row safely.
  - If the token number does not match, that means some other transaction has updated the row. So, in this case, the update fails and the client must retry again.
- More than one transaction can read the same row at the same time. That is totally fine.
  
<img width="900" alt="image" src="https://github.com/user-attachments/assets/200fc1d7-4645-4235-aa5c-d4a67ffffb29" />

## Codes
### Schema - adding a `version` column
```sql
CREATE TABLE accounts (
    id       SERIAL PRIMARY KEY,
    balance  INTEGER NOT NULL,
    version  INTEGER NOT NULL DEFAULT 1
);
```
### Making sure the version matches
```sql
-- You read version=1. Now you try to write:
UPDATE accounts
SET    balance = 150,
       version = version + 1
WHERE  id = 42
  AND  version = 1;   -- <-- this is the guard

-- If 0 rows updated → someone else changed it → retry
```

### JPA code
- Add the `@Version` annotation to a field in your entitiy. Hibernate will automatically manage incrementing this field on every update.
- If another transaction modified the row first, the version in the database will no longer match, returning `0` modified rows. Hibernate then throws an `OptimisticLockException`
```java
@Entity
public class Account {
    @Id
    private Long id;
    
    private int balance;

    @Version          // ← Hibernate manages this automatically
    private int version;
}

// Hibernate throws OptimisticLockException if the version mismatches
try {
    account.setBalance(account.getBalance() + 50);
    entityManager.merge(account);
    entityManager.getTransaction().commit();
} catch (OptimisticLockException e) {
    // Re-read and retry
}
```


