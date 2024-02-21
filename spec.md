
# Feature Proposal: Toggle Between Live and Demo

## Current Design Implementation

### User Keys

#### Key Generation

The current implementation stores user keys in the settings table, with the following schema:

```graphql
model settings {
  id             Int      @id @default(autoincrement())
  user           user     @relation(fields: [user_id], references: [id])
  user_id        Int      @unique
  private_key    String   @unique
  public_key     String   @unique
  webhook_url    String?
  callback_url   String?
  created_at     DateTime @updatedAt @db.Date
  updated_at     DateTime @updatedAt @db.Date
}
```

Users are issued keys tied to their live environment, determined by the `GATEWAY_ENV` environment variable (`process.env.GATEWAY_ENV`), which can either be LIVE or TEST.

### sample code 

```
async generateKey(type: ApiKeysEnum): Promise<string> {
    let generatedKey = '';

    switch (type) {
      case ApiKeysEnum.PRIVATE_KEY:
        generatedKey = `${process.env.GATEWAY_ENV}_PRK_${uid(50)}`;
        do {
          generatedKey = `${process.env.GATEWAY_ENV}_PRK_${uid(50)}`;
        } while ((await this.findSettingsWithKey(generatedKey, type)) !== null);
        break;
      case ApiKeysEnum.PUBLIC_KEY:
        generatedKey = `${process.env.GATEWAY_ENV}_PUK_${uid(50)}`;
        do {
          generatedKey = `${process.env.GATEWAY_ENV}_PUK_${uid(50)}`;
        } while ((await this.findSettingsWithKey(generatedKey, type)) !== null);
        break;
    }
    return generatedKey;
  }

// and example of a generated public key  TEST_PUK_0a85ebdc436bf8f0f7c7d81ab16f89f5f301cbc1feeb6070bc
```
So when a query is made to generate API keys on the test enviroment they are persisted as random strings prefixed with  TEST_PUK`



#### Authorization Logic

The system utilizes JWT for authorization and NestJS Guards for protecting routes. Guards act as middleware components that are executed before a request reaches the route handler, allowing or denying access based on predefined conditions.

#### Getting the Currently Logged In User

Upon successful authorization, the user's payload is stored in the request object. The `@LoggedInUser` custom parameter decorator extracts user properties stored in the request object.

### Use Cases

#### GET WEB_TRANSACTIONS

- User submits credentials and generates a token.
- The `getTransactions` endpoint, being a protected route, passes through the AuthGuard for token verification.
- User ID is extracted using the `@LoggedInUser` decorator and utilized in the service's business logic to query user transactions.

#### PAY_WITH_CARD

The current implementation allows integration of the checkout page via plugins, enabling non-authenticated access to the payment gateway.

## Proposed Design

### Dynamic Key Generation

A schema enhancement is proposed to enable users to toggle seamlessly between live and demo environments. This involves storing both test and live keys for each user.

```graphql
model settings {
  id               Int     @id @default(autoincrement())
  user             user    @relation(fields: [user_id], references: [id])
  user_id          Int     @unique
  private_key      String  @unique
  public_key       String  @unique
  private_key_test String  @unique 
  public_key_test  String  @unique
  webhook_url      String?
  callback_url     String?
  created_at       DateTime @updatedAt @db.Date
  updated_at       DateTime @updatedAt @db.Date
}
```

### Environment-Based Key Exposure

Users with a `kyc_verified` status can toggle between demo and live environments. In the demo environment, test keys are exposed, while in the live environment, live keys are exposed. Users without KYC verification are restricted to the demo environment and are only exposed to test keys.

### Persisting Transactions

Separate tables for test transactions ,payouts and wallet are proposed to mirror the existing live tables. All accounts created in the test environment would be stored in these tables, facilitating lookup operations

### Provider Services and Environment Variables

Environment variables for provider test keys would be added, with business logic in the demo environment utilizing these test keys.

### Custom Endpoints

Custom endpoints (`api/v1/dev/payouts`, `api/v1/dev/transactions`, `api/v1/dev/settings`) would exclusively be available in the demo environment.

---

This proposed design aims to enhance flexibility and security while providing a seamless experience for users toggling between live and demo environments.
