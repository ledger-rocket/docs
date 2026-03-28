# Numscript Examples

Practical examples demonstrating Numscript patterns for real-world financial applications.

## Example 1: Simple Transfer

```numscript
send [USD/2 1000] (
  source = @world
  destination = @users:001
)
```

Introduces $10.00 into the ledger for user 001.

## Example 2: Ride-Sharing Platform (RideShare)

> Source: <https://docs.formance.com/modules/ledger/tutorial>

### Step 1: Rider Books a Ride

Record the estimated payment from the rider:

```numscript
send [USD/2 2000] (
  source = @world
  destination = @rider:xx:ride:yy:payment
)
```

### Step 2: Confirm Payment

Transfer from rider payment to ride main account:

```numscript
send [USD/2 2000] (
  source = @rider:xx:ride:yy:payment
  destination = @ride:yy:main
)
```

### Step 3: Split Payment (Driver + Fees)

Split funds between driver earnings and service fees:

```numscript
send [USD/2 2000] (
  source = @ride:yy:main
  destination = {
    10% to @ride:yy:fees
    remaining to @driver:zz:main
  }
)
```

### Step 4: Driver Withdrawal

Transfer all driver earnings to external world:

```numscript
send [USD/2 *] (
  source = @driver:zz:main
  destination = @world
)
```

### Chart of Accounts

| Account | Purpose |
|---------|---------|
| `@world` | External funds entering/exiting the ledger |
| `rider:xx:ride:yy:payment` | Estimated rider charges for a ride |
| `ride:yy:main` | Amount charged for ride |
| `driver:zz:ride:yy` | Amount owed to driver after verification |
| `ride:yy:fees` | Service fees collected |
| `driver:zz:main` | Total driver earnings pending payout |

## Example 3: Marketplace Payment with Splits

```numscript
// Capture payment
send [USD/2 599] (
  source = @world
  destination = @payments:001
)

// Move to clearing
send [USD/2 599] (
  source = @payments:001
  destination = @rides:0234
)

// Split among driver, charity, and platform
send [USD/2 599] (
  source = @rides:0234
  destination = {
    85% to @drivers:042
    remaining to {
      10% to @charity
      remaining to @platform:fees
    }
  }
)
```

Result: 510 to drivers, 9 to charity, 80 to platform fees.

## Example 4: Coupon with Cap

```numscript
send [USD/2 29900] (
  source = {
    10% from {
      max [USD/2 2000] from @coupons:FALL24
      @users:1234
    }
    remaining from @users:1234
  }
  destination = @payments:4567
)
```

10% discount from a coupon capped at $20.00. Customer pays the rest.

## Example 5: Food Delivery Platform (Uber Eats Clone)

> Source: <https://dev.to/formance/building-an-uber-eats-clone-38j8>

### Payment Collection

```numscript
send [USD/2 5900] (
  source = @world
  destination = @payments:stripe
)
```

### Move to Order Account

```numscript
send [USD/2 5900] (
  source = @payments:stripe
  destination = @orders:001
)
```

### Commission Split (15% platform fee)

```numscript
vars {
  monetary $restaurant_amount
  monetary $rider_amount
  monetary $commission
}

send $restaurant_amount (
  source = @orders:001
  destination = @restaurants:001
)

send $rider_amount (
  source = @orders:001
  destination = @riders:001
)

send $commission (
  source = @orders:001
  destination = @platform:fees
)
```

### Coupon/Marketing Budget

```numscript
// Allocate marketing budget
send [USD/2 15000] (
  source = @world
  destination = @coupons:campaign_001
)

// Apply coupon to customer
send [USD/2 1900] (
  source = @coupons:campaign_001
  destination = @customers:001
)
```

## Example 6: Metadata-Driven Commission

```numscript
vars {
  account  $merchant
  portion  $commission = meta($merchant, "commission_rate")
  monetary $amount
}

send $amount (
  source = @orders:current
  destination = {
    $commission to @platform:fees
    remaining to $merchant
  }
)

set_tx_meta("merchant", $merchant)
set_tx_meta("commission_rate", $commission)
```

## Example 7: Payout with Minimum Balance

```numscript
save [USD/2 5000] from @merchants:1234

send [USD/2 *] (
  source = @merchants:1234
  destination = @payouts:batch_001
)
```

Sends all available balance except the protected $50.00 minimum.

## Example 8: Multi-Currency Exchange

```numscript
vars {
  monetary $usd_amount
  monetary $eur_amount
}

send $usd_amount (
  source = @users:001
  destination = @exchange:usd_eur
)

send $eur_amount (
  source = @exchange:usd_eur
  destination = @users:001
)
```

## Example 9: Overdraft Recovery

```numscript
#![feature("experimental-overdraft-function")]

vars {
  monetary $debt = overdraft(@users:001, USD/2)
}

send $debt (
  source = @world
  destination = @users:001
)
```

Automatically tops up an account that has gone into overdraft.

## Example 10: BTC Penny Split

```numscript
send [BTC/8 943] (
  source = @users:1234
  destination = {
    50% to @users:4567
    50% to @users:7890
  }
)
```

Result: 472 satoshis to first account, 471 to second (remainder goes top-to-bottom).
