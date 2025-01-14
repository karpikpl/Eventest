# Eventest

The purpose of this library is to serve as a demonstration of how to run end to end tests for Event Driven Systems.

When creating web applications that have asynchronous backend systems it can be very challenging to test those backends.
In an e-commerce example: It would be useful to test that putting a new Order into the Order Service causes:
   
1. Website sends a NewReservationRequest through a POST to the Reservation Service
2. Reservation Service generates a ```NewReservationReceived``` event.
3. Reservation Service issues ```TakePayment``` command to the Payment Service
4. Payment Service issues ```Payment Taken Event```
5. Reservation Service issues a ```ReservationConfirmed``` event


## Example of use of this library:

The example test scripts have been written to run against the application thats hosted at [https://github.com/ciaranodonnell/eventest.testservices](https://github.com/ciaranodonnell/eventest.testservices).
This test application is a container that has REST endpoints and publishes and subscribes to Azure Service Bus.

To write these tests we import our Eventtest package
``` TypeScript
import {Broker, Subscription, ReceiveResult, http, MassTransitMessageEncoder, MessageEncoder}  from 'Eventest';
import { AzureServiceBusTester } from 'Eventest.ServiceBus'

```

We then bring in ```mocha``` and ```chai``` to do tests, plus some other utilities to get them working. 
``` TypeScript
import 'mocha';
import { expect } from 'chai';

// This is a cool library to work with dates and times
import moment from 'moment';

// This allows us to await timeouts
import {promisify } from 'util';
const delay = promisify(setTimeout);

// simple setup of environment variables for the tests
import * as dotenv from 'dotenv';
// Load the .env file if it exists
dotenv.config();
```

Create a `.env` file with following environment variables:
```
SERVICEBUS_CONNECTION_STRING="Endpoint=sb://<XXXX>.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=<XXXXXXXX>"
SUBMIT_RESERVATION_SERVICE_ENDPOINT="http://localhost/Reservation"
GET_RESERVATION_SERVICE_ENDPOINT="http://localhost/Reservation"
```

Service Bus should be configured with following topics:
* newreservationreceived
* takepayment
* paymenttaken
* paymentrejected
* reservationrejected


Once we have that setup, we can write our actual test:

``` typescript
 
describe('Submitting NewReservationRequest', async () => {

    // create variables for our test state
    let test: Broker;

    let newReservationReceivedSubscription: Subscription;
    let takePaymentSubscription: Subscription;
    let paymentTakenSubscription: Subscription;
    let reservationConfirmedSubscription: Subscription;

    const testReservationId = 123;

    before(async () => {
        // runs once before the first test in this block

        // Create a Service Bus connection for this test
        test = new AzureServiceBusTester(
            process.env.SERVICEBUS_CONNECTION_STRING ?? '',
            new MassTransitMessageEncoder()
        );

        // Subscribe to the topic first so we dont miss the messages
        newReservationReceivedSubscription = await test.subscribeToTopic('newreservationreceived');
        takePaymentSubscription = await test.subscribeToTopic('takepayment');
        paymentTakenSubscription = await test.subscribeToTopic('paymenttaken');
        reservationConfirmedSubscription = await test.subscribeToTopic('reservationconfirmed');

        // give it a couple of seconds to make sure the subscriptions are active
        delay(2000);
    });

    it('should get OK status', async () => {
        const svcResponse = await http.postToService(
            process.env.SUBMIT_RESERVATION_SERVICE_ENDPOINT ?? '',
         // This is the payload to send to the service:
            {
                RequestCorrelationId: test.testUniqueId,
                ReservationId: testReservationId,
                StartDate: moment().format('YYYY-MM-DDTHH:mm:ss'),
                EndDate: moment().add('2','days').format('YYYY-MM-DDTHH:mm:ss'),
                GuestId: 123
            });
        // test that we got a 200 success
        expect(svcResponse.statusCode).equal(200);
    });

    it('should publish NewReservationEvent', async () => {
        // wait up to 2 seconds to receive a message on our subscription
        const receivedMessage = await newReservationReceivedSubscription.waitForMessage(2000);
        // test we got a message
        expect(receivedMessage.didReceive).equal(true);
        // test the reservation Id matches
        expect(receivedMessage.getMessageBody().reservationId).equal(testReservationId);
    });

    it('should return the Reservation', async () => {
        const result = await http.getFromService(
            (process.env.GET_RESERVATION_SERVICE_ENDPOINT ?? '')
        + '?reservationId=' + testReservationId);
        expect(result.success).equal(true);
    });

    it('should publish Take Payment Command', async () => {
        const receivedMessage = await takePaymentSubscription.waitForMessage(2000);
        expect(receivedMessage.didReceive).equal(true);
    });

    it('should publish Payment Taken Event', async () => {
        const receivedMessage = await paymentTakenSubscription.waitForMessage(2000);
        expect(receivedMessage.didReceive).to.equal(true);
    });

    it('should publish ReservationConfirmed event', async () => {
        const receivedMessage = await reservationConfirmedSubscription.waitForMessage(2000);
        expect(receivedMessage.didReceive).equal(true);
    });

    it('should return the Reservation as State=Confirmed', async () => {
        const result = await http.getFromService(
            (process.env.GET_RESERVATION_SERVICE_ENDPOINT ?? '')
            + '?reservationId=' + testReservationId);
        // test we got a 200 level response
        expect(result.success).equal(true);
        // test that the object in the body had a field called status with a value = 'Confirmed'
        expect(result.body.Status).equal('Confirmed');

    });

    // Clean up after all the tests
    after(async () => {
      // this removes all the subscriptions we made
      test.cleanup();
    });
});

```

## Example output:


### Passing Tests:
When running locally on a console window (shown here in powershell):
![Screenshot of Passing Tests run in Powershell](Eventest/docs/PassingTests.png)

It's also possible to run these as a CI/CD pipeline, perhaps after you've done an automated deployment of your application.
This is a screen shot of run summary from Azure Devops:
![Screenshot of Test Run summary from Azure DevOps](Eventest/docs/PassingTestsInAzDo.png)

and a list of tests:
![Screenshot of test list in Azure DevOps](Eventest/docs/PassingTestsListInAzDo.png)
### Failing Test

![Screenshot of Passing Tests](Eventest/docs/FailingTest.png)
