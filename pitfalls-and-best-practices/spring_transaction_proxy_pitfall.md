# Spring Transaction Proxy Pitfall: Self-Invocation Problem

## The Problem: Self-Invocation and Proxy Bypass

When you call a `@Transactional` method from within the same class, Spring's AOP proxy mechanism is bypassed, meaning the transaction annotation is completely ignored.

## Why This Happens

Spring uses **proxy-based AOP** for transaction management. When you inject a Spring bean, you're actually getting a proxy object that wraps your actual service. This proxy intercepts method calls and applies cross-cutting concerns like transactions.

However, when you call a method from within the same class, you're calling it directly on `this`, bypassing the proxy entirely.

```java
@Service
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    public void createUsers(List<User> users) {
        for (User user : users) {
            // ❌ This bypasses the proxy - @Transactional is ignored!
            this.createUser(user);
        }
    }
    
    @Transactional
    public void createUser(User user) {
        userRepository.save(user);
        // If this fails, no rollback will happen because 
        // the transaction was never started!
    }
}
```

## Real-World Example: The Banking Service Pitfall

```java
@Service
public class BankingService {
    
    @Autowired
    private AccountRepository accountRepository;
    
    @Autowired
    private TransactionLogRepository transactionLogRepository;
    
    // This method is NOT transactional
    public void transferMoney(Long fromAccountId, Long toAccountId, BigDecimal amount) {
        Account fromAccount = accountRepository.findById(fromAccountId).orElseThrow();
        Account toAccount = accountRepository.findById(toAccountId).orElseThrow();
        
        if (fromAccount.getBalance().compareTo(amount) < 0) {
            throw new InsufficientFundsException();
        }
        
        // ❌ These calls bypass the proxy - no transaction protection!
        this.debitAccount(fromAccountId, amount);
        this.creditAccount(toAccountId, amount);
        this.logTransaction(fromAccountId, toAccountId, amount);
    }
    
    @Transactional
    public void debitAccount(Long accountId, BigDecimal amount) {
        Account account = accountRepository.findById(accountId).orElseThrow();
        account.setBalance(account.getBalance().subtract(amount));
        accountRepository.save(account);
    }
    
    @Transactional  
    public void creditAccount(Long accountId, BigDecimal amount) {
        Account account = accountRepository.findById(accountId).orElseThrow();
        account.setBalance(account.getBalance().add(amount));
        accountRepository.save(account);
        
        // If this fails, the debit above won't be rolled back!
        if (account.getBalance().compareTo(new BigDecimal("1000000")) > 0) {
            throw new SuspiciousTransactionException();
        }
    }
    
    @Transactional
    public void logTransaction(Long fromId, Long toId, BigDecimal amount) {
        TransactionLog log = new TransactionLog(fromId, toId, amount);
        transactionLogRepository.save(log);
    }
}
```

**Problem:** If `creditAccount()` fails, the `debitAccount()` operation won't be rolled back because each method runs in its own separate transaction (or no transaction at all).

## Solutions

### Solution 1: Move the @Transactional to the Calling Method

```java
@Service
public class BankingService {
    
    @Autowired
    private AccountRepository accountRepository;
    
    @Autowired
    private TransactionLogRepository transactionLogRepository;
    
    // ✅ Single transaction covers all operations
    @Transactional
    public void transferMoney(Long fromAccountId, Long toAccountId, BigDecimal amount) {
        Account fromAccount = accountRepository.findById(fromAccountId).orElseThrow();
        Account toAccount = accountRepository.findById(toAccountId).orElseThrow();
        
        if (fromAccount.getBalance().compareTo(amount) < 0) {
            throw new InsufficientFundsException();
        }
        
        // These are now simple method calls within the same transaction
        this.debitAccount(fromAccountId, amount);
        this.creditAccount(toAccountId, amount);
        this.logTransaction(fromAccountId, toAccountId, amount);
    }
    
    // Remove @Transactional - these are now private helper methods
    private void debitAccount(Long accountId, BigDecimal amount) {
        Account account = accountRepository.findById(accountId).orElseThrow();
        account.setBalance(account.getBalance().subtract(amount));
        accountRepository.save(account);
    }
    
    private void creditAccount(Long accountId, BigDecimal amount) {
        Account account = accountRepository.findById(accountId).orElseThrow();
        account.setBalance(account.getBalance().add(amount));
        accountRepository.save(account);
        
        if (account.getBalance().compareTo(new BigDecimal("1000000")) > 0) {
            throw new SuspiciousTransactionException();
        }
    }
    
    private void logTransaction(Long fromId, Long toId, BigDecimal amount) {
        TransactionLog log = new TransactionLog(fromId, toId, amount);
        transactionLogRepository.save(log);
    }
}
```

### Solution 2: Self-Injection Pattern

```java
@Service
public class BankingService {
    
    @Autowired
    private AccountRepository accountRepository;
    
    @Autowired
    private TransactionLogRepository transactionLogRepository;
    
    // ✅ Inject self to get the proxy
    @Autowired
    private BankingService self;
    
    public void transferMoney(Long fromAccountId, Long toAccountId, BigDecimal amount) {
        Account fromAccount = accountRepository.findById(fromAccountId).orElseThrow();
        Account toAccount = accountRepository.findById(toAccountId).orElseThrow();
        
        if (fromAccount.getBalance().compareTo(amount) < 0) {
            throw new InsufficientFundsException();
        }
        
        // ✅ Call through the proxy to activate @Transactional
        self.debitAccount(fromAccountId, amount);
        self.creditAccount(toAccountId, amount);
        self.logTransaction(fromAccountId, toAccountId, amount);
    }
    
    @Transactional
    public void debitAccount(Long accountId, BigDecimal amount) {
        Account account = accountRepository.findById(accountId).orElseThrow();
        account.setBalance(account.getBalance().subtract(amount));
        accountRepository.save(account);
    }
    
    @Transactional(propagation = Propagation.REQUIRED)
    public void creditAccount(Long accountId, BigDecimal amount) {
        Account account = accountRepository.findById(accountId).orElseThrow();
        account.setBalance(account.getBalance().add(amount));
        accountRepository.save(account);
        
        if (account.getBalance().compareTo(new BigDecimal("1000000")) > 0) {
            throw new SuspiciousTransactionException();
        }
    }
    
    @Transactional(propagation = Propagation.REQUIRED)
    public void logTransaction(Long fromId, Long toId, BigDecimal amount) {
        TransactionLog log = new TransactionLog(fromId, toId, amount);
        transactionLogRepository.save(log);
    }
}
```

### Solution 3: Using AopContext (Requires AspectJ)

```java
@Service
@EnableAspectJAutoProxy(exposeProxy = true)
public class BankingService {
    
    public void transferMoney(Long fromAccountId, Long toAccountId, BigDecimal amount) {
        // ✅ Get the current proxy
        BankingService proxy = (BankingService) AopContext.currentProxy();
        
        // Validation logic...
        
        proxy.debitAccount(fromAccountId, amount);
        proxy.creditAccount(toAccountId, amount);
        proxy.logTransaction(fromAccountId, toAccountId, amount);
    }
    
    @Transactional
    public void debitAccount(Long accountId, BigDecimal amount) {
        // Implementation...
    }
    
    // Other methods...
}
```

### Solution 4: Split into Separate Services (Best Practice)

```java
@Service
public class AccountService {
    
    @Autowired
    private AccountRepository accountRepository;
    
    @Transactional
    public void debitAccount(Long accountId, BigDecimal amount) {
        Account account = accountRepository.findById(accountId).orElseThrow();
        account.setBalance(account.getBalance().subtract(amount));
        accountRepository.save(account);
    }
    
    @Transactional
    public void creditAccount(Long accountId, BigDecimal amount) {
        Account account = accountRepository.findById(accountId).orElseThrow();
        account.setBalance(account.getBalance().add(amount));
        accountRepository.save(account);
        
        if (account.getBalance().compareTo(new BigDecimal("1000000")) > 0) {
            throw new SuspiciousTransactionException();
        }
    }
}

@Service
public class TransactionLogService {
    
    @Autowired
    private TransactionLogRepository transactionLogRepository;
    
    @Transactional
    public void logTransaction(Long fromId, Long toId, BigDecimal amount) {
        TransactionLog log = new TransactionLog(fromId, toId, amount);
        transactionLogRepository.save(log);
    }
}

@Service
public class BankingService {
    
    @Autowired
    private AccountService accountService;
    
    @Autowired
    private TransactionLogService transactionLogService;
    
    @Autowired
    private AccountRepository accountRepository;
    
    // ✅ This orchestrates the transaction across services
    @Transactional
    public void transferMoney(Long fromAccountId, Long toAccountId, BigDecimal amount) {
        Account fromAccount = accountRepository.findById(fromAccountId).orElseThrow();
        Account toAccount = accountRepository.findById(toAccountId).orElseThrow();
        
        if (fromAccount.getBalance().compareTo(amount) < 0) {
            throw new InsufficientFundsException();
        }
        
        // ✅ These calls go through proxies and respect @Transactional
        accountService.debitAccount(fromAccountId, amount);
        accountService.creditAccount(toAccountId, amount);
        transactionLogService.logTransaction(fromAccountId, toAccountId, amount);
    }
}
```

## Common Propagation Gotchas

### REQUIRES_NEW Pitfall

```java
@Service
public class OrderService {
    
    @Autowired
    private OrderService self; // Self-injection
    
    @Transactional
    public void processOrder(Order order) {
        // Main transaction
        orderRepository.save(order);
        
        // ❌ This starts a NEW transaction that commits independently
        self.logOrderProcessing(order.getId());
        
        // If this fails, the log above won't be rolled back!
        if (order.getAmount().compareTo(BigDecimal.ZERO) <= 0) {
            throw new InvalidOrderException();
        }
    }
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logOrderProcessing(Long orderId) {
        OrderLog log = new OrderLog(orderId, "Processing started");
        orderLogRepository.save(log);
    }
}
```

## Best Practices

1. **Prefer service composition over self-injection** - Split functionality into logical services
2. **Keep transactions at the service boundary** - Don't scatter `@Transactional` on every method
3. **Use `@Transactional(readOnly = true)`** for read-only operations
4. **Test transaction boundaries** - Write integration tests that verify rollback behavior
5. **Monitor transaction logs** - Enable transaction logging to catch these issues

```properties
# Enable transaction logging
logging.level.org.springframework.transaction.interceptor=TRACE
logging.level.org.springframework.transaction.support=DEBUG
```

## Detection and Testing

```java
@SpringBootTest
@Transactional
@Rollback
class BankingServiceTest {
    
    @Test
    void shouldRollbackOnFailure() {
        assertThatThrownBy(() -> 
            bankingService.transferMoney(1L, 2L, new BigDecimal("1000000"))
        ).isInstanceOf(SuspiciousTransactionException.class);
        
        // Verify that no changes were persisted
        Account fromAccount = accountRepository.findById(1L).orElseThrow();
        Account toAccount = accountRepository.findById(2L).orElseThrow();
        
        assertThat(fromAccount.getBalance()).isEqualTo(originalFromBalance);
        assertThat(toAccount.getBalance()).isEqualTo(originalToBalance);
    }
}
```

Remember: **Always call `@Transactional` methods through Spring proxies, never through `this`**. When in doubt, split your services or move the transaction boundary to the calling method.

---

## Key Takeaway

> **`@Transactional` methods must be called from another bean (the proxy) to work.** Self-invocation (calling a transactional method from within the same class) bypasses the proxy and ignores the transaction. The best solution is to refactor your code to have a single, public, transactional method that orchestrates private, non-transactional helper methods.

---

## See Also

-   [Improper Transaction Management](./jpa_transaction_management.md)
