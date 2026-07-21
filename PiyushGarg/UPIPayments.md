# System Design of UPI Payments

Source video: [System Design of UPI Payments](https://www.youtube.com/watch?v=fqySz1Me2pI) (Hindi, about 25 minutes)

These notes are written in simple English based on the Hindi transcript of the video. Definitions, examples, and diagrams have been added to make each concept easier to follow. UPI (India's largest payment infrastructure) lets you transfer money to almost anyone instantly, using just a simple ID. This video looks at the high-level system design behind how that actually works, based on publicly available blogs and articles, since UPI's own internal, low-level implementation details are not publicly documented.

---

## 1. Before UPI: Traditional Bank-to-Bank Transfers

Before understanding UPI, it helps to understand how digital money transfers worked before it existed.

At the center of any digital transfer are **banks** (for example, ICICI, HDFC, or SBI). All banks in India are regulated institutions, and whatever infrastructure or technology they use is regulated by the **RBI (Reserve Bank of India)**.

**To transfer money the traditional way, you needed several pieces of information:**

```
- Account Number   (a unique identifier for the account)
- Bank Name         (e.g. ICICI Bank)
- Branch Code       (since a single bank has many branches)
- IFSC Code         (a unique code identifying the specific bank branch)
```

Together, these details uniquely identify exactly which account, at which bank, at which branch, money should be sent to. Without all of these details being correct, a transfer would simply fail or get reverted.

**Traditional transfer mechanisms included:**

- **IMPS (Immediate Payment Service):** Used for smaller amounts (say, ₹1,000 to ₹2,000) where the transfer needed to reflect instantly.
- **NEFT (National Electronic Funds Transfer):** Could handle larger amounts (say, ₹50,000 to ₹1,00,000). Historically (before December 2019), NEFT was not instant: it settled in fixed hourly batches only during banking hours on working days, so a transfer could take a couple of hours, or even until the next business day, to reflect in the recipient's account, which is why people would often share a UTR number or a screenshot as temporary proof of payment. Since December 2019, the RBI made NEFT available 24x7x365, settling in half-hourly batches, so transfers today typically reflect within about 30 minutes.
- **RTGS (Real Time Gross Settlement):** Used for very large amounts (with a regulatory minimum of ₹2,00,000 per transaction, and no upper limit), and was genuinely real-time, reflecting in the recipient's account as soon as the transfer was made. RTGS has also been available 24x7x365 since December 2020.

These traditional systems required a lot of detailed information and were often not instant. **UPI (Unified Payment Interface)** was built specifically to simplify this entire experience: open an app, enter a simple ID, and the transfer is done, instantly.

---

## 2. The Core Infrastructure: NPCI

> **Definition:** NPCI (National Payments Corporation of India) is the central infrastructure that powers UPI. NPCI itself is not a payment gateway or a payment provider that end users interact with directly. It's the secure backend network that makes UPI transfers possible, and it operates as a closed, trusted network.

**A critical detail: NPCI's APIs are not publicly accessible.** You (or any random app or company) cannot connect to NPCI's network directly. NPCI only communicates with a specific, trusted set of banks.

```
NPCI's Trusted Network (a closed, secure network)
   |--> ICICI Bank    (trusted)
   |--> HDFC Bank      (trusted)
   |--> SBI            (trusted)
   |--> Yes Bank        (trusted)
   |--> ...(and other trusted banks)
```

No other user, system, or company can call NPCI's network directly. Only these trusted banks can.

---

## 3. How Everyday Apps (Google Pay, PhonePe) Connect to NPCI

Since apps like Google Pay, PhonePe, and Paytm are not banks themselves, they cannot talk to NPCI directly either.

> **Definition:** A Customer PSP (Payment Service Provider) is the app a user actually interacts with, such as Google Pay, PhonePe, or Paytm. Its job is to provide the user interface (scanning QR codes, entering payment details, and so on).

Since these apps can't connect to NPCI's closed network on their own, **each app partners with a specific bank**, which acts as its bridge into NPCI's trusted network.

```
Customer's App (Customer PSP)         Its Partner Bank (bridge into NPCI)
   PhonePe                    -->        Yes Bank
   Google Pay                 -->        ICICI Bank
```

This partner bank is what actually submits the payment request to NPCI's network on the app's behalf, since only trusted banks are allowed to talk to NPCI directly.

---

## 4. VPA: Virtual Payment Address

One of UPI's biggest simplifications is replacing the old "account number + bank name + IFSC code" combination with something far simpler.

> **Definition:** A VPA (Virtual Payment Address) is a simple, human-readable identifier used in UPI, in the form `username@handle` (for example, `piyush@icici` or `someone@okhdfc`). It replaces the need to remember a full account number, bank name, and IFSC code.

A VPA is easy to remember and easy to share. When you scan a QR code to pay someone, the QR code is really just a convenient, error-proof way of entering that person's VPA, instead of typing it manually and risking a typo.

---

## 5. Walking Through a Complete UPI Transaction

Let's trace a full example: a user paying ₹1,000 to Piyush (`piyush@icici`) using PhonePe.

**Step 1: The user creates a payment intent.**

```
User opens PhonePe (their Customer PSP)
   |
   |  scans a QR code or manually enters the recipient's VPA
   v
An "intent" is created:
   From:   user1@yesbank
   To:     piyush@icici
   Amount: 1000
```

**Step 2: The app forwards the request to its own partner bank.**

```
PhonePe
   |
   |  PhonePe's partner bank is Yes Bank
   v
Yes Bank receives the payment request
```

**Step 3: The partner bank submits the request to NPCI.**

```
Yes Bank
   |
   |  Yes Bank is a trusted member of NPCI's network
   v
NPCI accepts the request, since it's coming from a trusted source
```

**Step 4: NPCI checks with the payer's bank for balance and authentication.**

```
NPCI
   |
   |  "Does user1's account have ₹1000 available?"
   v
Yes Bank responds: "Yes, the balance is available."
   |
   |  Yes Bank requests authentication
   v
PhonePe shows the user the UPI PIN entry screen
   |
   |  user enters their PIN
   v
Authentication is confirmed
```

**Step 5: The payer's bank debits the amount.**

```
NPCI instructs Yes Bank: "Debit ₹1000 from user1's account."
   |
   |  Yes Bank accepts, since the instruction is coming from the trusted NPCI network
   v
₹1000 is debited from user1's account
```

**Step 6: NPCI instructs the payee's bank to credit the amount.**

```
NPCI instructs ICICI Bank: "Credit ₹1000 to piyush@icici's account."
   |
   |  ICICI Bank accepts, since the instruction is coming from the trusted NPCI network
   v
₹1000 is credited to Piyush's account
```

**Step 7: Both sides acknowledge, and the transaction completes.**

```
Yes Bank  -->  acknowledges the debit was successful
ICICI Bank -->  acknowledges the credit was successful
   |
   |  only once BOTH acknowledgements are received...
   v
The transaction is marked as COMPLETE

ICICI Bank then notifies Google Pay (Piyush's Customer PSP)
   -> "₹1000 has been credited to Piyush's account"
   -> Google Pay shows Piyush a notification: "Received ₹1000"
```

```
Full flow, end to end:

User -> PhonePe (Customer PSP) -> Yes Bank (Partner Bank) -> NPCI
                                                                |
                                          +---------------------+---------------------+
                                          |                                           |
                                          v                                           v
                                   Yes Bank (debit)                          ICICI Bank (credit)
                                          |                                           |
                                          +-------------------+-----------------------+
                                                                |
                                                    both acknowledge successfully
                                                                |
                                                                v
                                                  ICICI Bank notifies Google Pay
                                                                |
                                                                v
                                                    Piyush sees "Received ₹1000"
```

---

## 6. Why Money Sometimes "Comes Back" After a Payment

You may have experienced this yourself: you pay a shopkeeper, the money is debited from your account, but the shopkeeper says the payment never arrived, and a little while later, the money returns to your account. This is a direct consequence of the acknowledgement step described above.

> **Explanation:** If, for example, ICICI Bank's server is too busy at that moment to process and acknowledge the incoming credit request, NPCI never receives that acknowledgement. Since the transaction only completes once **both** sides (debit and credit) successfully acknowledge, NPCI will treat a missing acknowledgement as a failure, and it will instruct the payer's bank to **credit the money back** to the original account. This is exactly why the money "returns" after a failed-looking payment: the debit happened, but the credit side didn't fully acknowledge in time, so the whole transaction gets rolled back to keep both sides consistent.

---

## 7. Key Participants in UPI (Terminology)

Based on the video's research into publicly available material (including a RazorPay blog on how UPI works), the key roles in any UPI transaction are:

| Role | Description |
|---|---|
| Payer PSP | The payment app used by the person paying (e.g. PhonePe, Google Pay) |
| Payee PSP | The payment app used by the person receiving the payment |
| Issuing Bank | The bank the money is being debited from (the payer's bank) |
| Acquiring Bank | The bank the money is being credited to (the payee's bank) |
| NPCI | The central, trusted infrastructure that routes and coordinates the entire transaction |

> **Example of real partner bank relationships mentioned in the video:** Google Pay has partnered with banks like Axis, ICICI, HDFC, and SBI. PhonePe has partnered with banks like Yes Bank, ICICI, and Axis. This is why a payment app needs a bank partnership in the first place: to get access into NPCI's otherwise closed network.

---

## 8. Push vs. Pull Transactions

The example above (a user actively paying someone) is what's known as a **push** transaction: the payer initiates and pushes money out.

> **Definition:** In a pull transaction, the request originates from the payee instead. For example, when you send someone a payment request ("please pay me ₹1000"), or when an online merchant requests payment during checkout, this works as a pull mechanism through NPCI's network: the customer's app makes a "pull" request to NPCI, and NPCI notifies the payer that there's a pending payment request awaiting their approval.

Both push and pull transactions ultimately flow through the same trusted NPCI network described above.

---

## 9. The Scale of NPCI's Infrastructure

It's worth appreciating just how demanding this system is under the hood. NPCI's infrastructure is estimated to handle on the order of **billions of transactions** flowing through it, with an enormous number of operations happening every single second. Building a real-time system at this scale, one that stays fault-tolerant and reliable under that kind of continuous load, is a genuinely impressive feat of engineering.

> **Note from the video's creator:** despite researching multiple blogs and articles (including material from RazorPay and Google Pay's own documentation) while preparing this video, detailed low-level implementation specifics of NPCI's internal systems (exact routing logic, tech stack, and so on) are not publicly available. What's covered here represents the high-level system design that can be pieced together from publicly available sources.

---

## Summary: How a UPI Transaction Works

| Step | What Happens |
|---|---|
| 1. Intent Creation | User enters or scans a VPA and specifies an amount inside their Customer PSP app (e.g. PhonePe) |
| 2. Forward to Partner Bank | The app forwards the request to its own partner bank (its bridge into NPCI) |
| 3. Submit to NPCI | The partner bank submits the payment request to NPCI's trusted, closed network |
| 4. Balance Check & Authentication | NPCI checks with the payer's bank for sufficient balance, then requires PIN authentication |
| 5. Debit | The payer's bank debits the amount, on NPCI's instruction |
| 6. Credit | NPCI instructs the payee's bank to credit the same amount |
| 7. Acknowledgement | Both banks must acknowledge success before the transaction is marked complete; a missing acknowledgement triggers a rollback |
| 8. Notification | The payee's bank notifies the payee's Customer PSP app, which shows a "payment received" notification |

**Key takeaway:** UPI's simplicity for end users (just a VPA and a PIN) is made possible by a closed, trusted network (NPCI) that only communicates with regulated, trusted banks. Every payment app (Google Pay, PhonePe, Paytm, and so on) must partner with a bank to gain entry into this network, and every transaction only completes once both the debit and the credit sides have been independently acknowledged as successful.
