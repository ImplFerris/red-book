# Mocking Transactions

One of the reasons I gave you the full test code earlier was to give you a clear picture of what we’re aiming for.

In this section, we will focus on understanding PinTx and DelayTx (these are just aliases for Transaction types). In embedded-hal-mock, a transaction is simply a list of expected actions that we think our code will perform.

Here are some examples:

- PinTx::set(PinState::High) means we expect the code to call set_high() on the pin.

- PinTx::get(PinState::Low) means we expect the code to check the pin's state, and we will return Low from the mock. If we want to return High, we can mock it with PinTx::get(PinState::High)

- DelayTx::delay_us(1) means we expect the code to wait for 1 microsecond.

## Understanding the Sequence

Let's take a closer look at this important sequence of code: it first sets the pin high, then waits for the pin to become high, and finally waits for it to become low.

```rust
Code                         |  Mock Expectations
-----------------------------|-------------------------------
pin.set_high().unwrap();     |  PinTx::set(PinState::High)
                             |
dht.wait_for_high().unwrap();|  PinTx::get(PinState::High)
                             |
dht.wait_for_low().unwrap(); |  PinTx::get(PinState::Low)
```

Here's the fun part: we can control how the mock behaves for wait_for_high() and wait_for_low(). It's up to us whether we return the expected pin state immediately or introduce a delay. Sounds a bit confusing? I know - it took me a while to fully get it too. Take your time, try out different setups, and experiment.

To help you understand better, let's walk through two scenarios: one where the expected state is returned immediately (the "happy path"), and another with a delay.

## Happy Path

In this scenario, we immediately return the expected pin state. That means there will be no delay.

```rust
Driver Code                      -->  Mock Response
-----------------------------------------------------------
pin.set_high().unwrap();         -->  PinTx::set(High)

dht.wait_for_high().unwrap();    -->  PinTx::get(High) ✅
                                     (No delay needed)

dht.wait_for_low().unwrap();     -->  PinTx::get(Low) ✅
                                     (No delay needed)
```

So, we will define the vector of expected pin transactions like this:

```rust
let mut expect = vec![];
expect.extend_from_slice(&[
    // pin setting high
    PinTx::set(PinState::High),
    // wait_for_high -> we give it high immediately
    PinTx::get(PinState::High),
    // wait_for_low -> we give it low immediately
    PinTx::get(PinState::Low),
]);

// Initialize the Pin Mock
let mut pin = PinMock::new(&expect);
``` 

Since in this happy path we immediately get what we want, there will be no delay (i.e. the helper function wait_for_state never calls delay_us), so we will keep the delay transactions vector empty.

```rust
let delay_transactions = vec![];

// Initialize the Delay Mock 
let mut delay = CheckedDelay::new(&delay_transactions);
```

 
Finally, we assert that both the pin and delay mocks completed as expected:

```rust
pin.done();
delay.done();
```

This test will pass without issue.

## Delayed Path

In a real scenario, the pin is not going to immediately change the state. As you already know, the DHT22 protocol uses timing to transmit bits, so we should expect some delays.

Now we slightly tweak our previous test. Instead of giving the High state immediately when wait_for_high runs, we simulate two failed attempts (i.e., we give it two Low states first). This means our function will poll the pin multiple times before finally getting the expected High.

```rust
let mut expect = vec![];
expect.extend_from_slice(&[
    // pin setting high
    PinTx::set(PinState::High),
    // wait_for_high
    PinTx::get(PinState::Low), // 1st try, not high
    PinTx::get(PinState::Low), // 2nd try, still not high
    PinTx::get(PinState::High), // expected state: high
    // wait_for_low -> we give it low immediately
    PinTx::get(PinState::Low),
]);
```

And here's how the sequence plays out:

```rust
Driver Code                      -->  Mock Response
-----------------------------------------------------------
pin.set_high().unwrap();         -->  PinTx::set(High)

dht.wait_for_high().unwrap();    -->  PinTx::get(Low) ❌ // we return Low, so driver delays
                                      DelayTx::delay_us(1)
                                 -->  PinTx::get(Low) ❌ // we still give Low, so driver delays again
                                      DelayTx::delay_us(1)
                                 -->  PinTx::get(High) ✅ // finally we return High

dht.wait_for_low().unwrap();     -->  PinTx::get(Low) ✅
                                     (No delay needed)
```

If you try to run the test with only the pin transactions updated, it will fail. That's because we're not giving the expected state immediately, so the driver will call delay; but our delay mock isn’t expecting that yet.

So let's fix that by updating the delay transactions:

```rust
let delay_transactions = vec![
    DelayTx::delay_us(1), 
    DelayTx::delay_us(1)
];
let mut delay = CheckedDelay::new(&delay_transactions);
```

With this change, the test will now pass. You can now start tweaking the values, try adding more delay entries or changing them  to see how the behavior changes. That's the best way to understand how it all fits together. 
