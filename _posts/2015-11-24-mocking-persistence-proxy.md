---
layout:     post
title:      Spring Component Mocking - A Hack
date:       2015-11-24
summary:    A writeup of a few aspects to consider when mocking Spring components. An example is given via the persistence layer.
categories: dynamic-proxy spring inject unit-testing
---

I'm not the biggest fan of using mock objects when implementing automated unit tests. The reason I say this is because very often, you find yourself meddling in the details of a method's implementation instead of focusing on simply testing it's interface. The extensive use of mock objects results in tests that are very brittle - i.e., any slight change to a method's implementation requires the updating of related test cases. For persistence related testing, I prefer setting up the data and running the tests in a real in-memory database like [HSQLDB](http://hsqldb.org/). There are those edge cases however that are easier to simulate using mock objects (e.g., all kinds of exceptions that can be thrown by the persistence layer). In a Spring/Hibernate environment, this can be achieved by injecting a mocked [DAO](https://en.wikipedia.org/wiki/Data_access_object) into a real service object.

The Scenario
---

Consider the following Spring based service implementation.

**The Interface:**

```java
public interface BankAPI {
    public void save(Account account) throws BankServiceException;
    public Account getAccount(String accountNumber) throws BankServiceException;
}
```

**The Concrete Class:**


```java 
@Service
public class BankAPIImpl implements BankAPI {

    @Autowired
    private BankDAO bankDAO;

    @Override
    @Transactional
    public void save(Account account) throws BankServiceException {

        try {
            bankDAO.save(account);
        }
        catch(DataIntegrityViolationException e) {

            throw new BankServiceException("An account with the given ID number already exists", e);
        }
        catch(CannotGetJdbcConnectionException e) {

            throw new BankServiceException("There seems to be a problem with the database. Please...", e);
        }
        catch(Exception e) {

            throw new BankServiceException("An unexpected error occurred. Please...", e);
        }
    }

    @Override
    public Account getAccount(String accountNumber) throws BankServiceException {

        try {
            return bankDAO.loadAccount(accountNumber);
        }
        catch (Exception e) {

            throw new BankServiceException("There was a problem loading the account.", e);
        }
    }
}
```

In the above example, we have a `BankAPI` service that provides access to a bank's infrastructure. The service method defined allows us to save a customer's account information.

The `BankAPI` service implementation delegates the persistence of the data to the DAO. Nothing special related to business logic happens here. The service implementation expects one of several outcomes; one of these is a scenario where a database connection can not be retrieved from the connection pool. To simulate this scenario, we can mock the DAO and have it throw a `CannotGetJdbcConnectionException` when the DAO's `save` method is called.

The Test and associated Hack
---

Below is the test class with all the juicy stuff fleshed out. The main test method `testSaveAccountFailNoConnection()` sets up the DAO mock object, expecting that `CannotGetJdbcConnectionException` will be thrown. After the mocking is setup, the service object is injected with the mocked DAO (the hack). The service object's `save()` method is then called, resulting in the expected database connection exception being thrown. At the end of the test, some clean up happens where the mocked DAO is replaced with the real one. 

```java 
public class TestBankAPI {

    private static final Log LOG = LogFactory.getLog(TestBankAPI.class);

    private BankAPI bankService = null;
    private BankDAO realBankDAO = null;
    private static ApplicationContext context = new ClassPathXmlApplicationContext("applicationContext.xml");

    @Before
    public void setup() {
        // do some setup
        bankService = (BankAPI) context.getBean("bankAPIImpl");
        realBankDAO = (BankDAO) context.getBean("bankDAOJPA");
    }

    @After
    public void tearDown() {
        // clean up
    }

    public static void injectField(Object object, String fieldName, Object fieldValue) throws TestException {
        try {
            // Get the member variable for which the value must be replaced
            Field field = object.getClass().getDeclaredField(fieldName);
            // Make sure it's accessible
            field.setAccessible(true);
            // Replace the value
            field.set(object, fieldValue);
        }
        catch (Exception e) {
            throw new TestException(e);
        }
    }

    @Test(expected = BankServiceException.class)
    public void testSaveAccountFailNoConnection() throws BankServiceException, TestException {

        // Setup mocking
        BankDAO dao = createMock(BankDAO.class);

        try {
            Account account = new Account("12345", "Joe Soap");
            dao.save(account);

            expectLastCall().andThrow(new CannotGetJdbcConnectionException(
                                            "Can't get connection.",
                                            new SQLException("Dummy SQL Exception.")));
            replay(dao);

            // Inject the DAO into the service object using reflection
            injectField(bankService, "bankDAO", dao);

            // Now call service method
            bankService.save(account);

        }
        catch (BankDataException e) {
            LOG.error(e);
            Assert.fail("Unexpected exception thrown during test: " + e.getMessage());
        }
        finally {
            // finally, restore the real DAO so other tests
            // expecting a real database can use it.
            injectField(bankService, "bankDAO", realBankDAO);
        }
    }

    @Test
    public void testSaveAccountForReal() {

        try {
            Account account = new Account("54321", "Joe Soap");

            // Now call service method
            bankService.save(account);
            Account newAccount = bankService.getAccount(account.getAccountNumber());
            Assert.assertEquals(account.getAccountNumber(), newAccount.getAccountNumber());
        }
        catch (BankServiceException e) {
            LOG.error(e.getMessage(), e);
            Assert.fail("Unexpected exception thrown during test: " + e.getMessage());
        }
    }
}
```

**The Gotcha (Spring)**

So here's the problem. The above injection won't work. If we run the above code, we'll find that the DAO on which `save` is invoked, won't be our mocked one; it will be the original one injected by Spring in the autowire process. When we call `save` on `bankService`, we must remember that by default, `bankService` is actually a [Cglib](https://github.com/cglib/cglib) dynamic proxy. So when we inject our mocked DAO into `bankService`, we're injecting it into the proxy and not the real component. Spring makes use of dynamic proxies (either JDK or Cglib) to implement AOP and interceptors. To make sure we're injecting the mocked DAO into the target `bankService` component, we need to change the `injectField()` method as follows:

```java 
public static void injectField(Class objectClass, Object object, String fieldName, Object fieldValue) throws Exception {
    	// Check if we're dealing with an 'advised' object (proxied), and if we are
    	// get the underlying target object
        if (object instanceof Advised)
            object = ((Advised)object).getTargetSource().getTarget();

        // Now we're sure we have the correct target object into which we inject 'fieldValue'
        Field field = objectClass.getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(object, fieldValue);
    }
```

The above inject code will ensure we inject our mocked DAO into the correct target component. By default JUnit runs the tests sequentially, so we can be sure that there are no side-effects impacting other tests that also use `bankService`.

I hope the above sample code saves you time and helps in some way to make your automated testing simpler. Get the code [here](https://github.com/ntobeko/proxy-inject-example).
