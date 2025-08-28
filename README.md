Binomial Option Pricing in Python
=================================

A personal Python project inspired by Sheldon Natenberg’s Option Volatility and Pricing: Advanced Trading Strategies and Techniques, Chapter 19 “Binomial Option Pricing”. This project was published to InsiderFinance Wire and my personal medium blog page and can be found [here](https://wire.insiderfinance.io/binomial-option-pricing-in-python-2547b0916f6f).

### Option Terminology

*   **Call Option** — The right to take a long position in an asset at a fixed price on or before a specified date.
*   **Put Option** — The right to take a long position in an asset.
*   **Underlying Spot Price** — The current market price of an underlying asset
*   **Strike Price** — The price at which the underlying will be delivered should the holder of an option choose to exercise their right to buy or sell
*   **Expiration Date** — The date on which the owner of an option must make the final decision to buy, in the case of a call, or to sell, in the case of a put.
*   **Exercise Style**:
    - A _European_ option can only be exercised at expiration.
    - An _American_ option can be exercised on any business day prior to expiration.
*   **Premium** — The price of an option.

### A Risk-Neutral World

We can generalise how an assets underlying price changes by assuming it either moves down or up over some time period. Suppose we have an underlying asset trading at 100, and in the future can take on one of two price, 120 and 90. There should be some probability of upward movement _p_ and downward movement 1-_p_ such that an investor will be indifferent as to whether he buys or sells. For an investor to be indifferent, the total expected value must be 100.

![captionless image](https://miro.medium.com/v2/resize:fit:812/format:webp/1*E5_mCkN6pv9WI1-DHqTt2A.png)
\
![captionless image](https://miro.medium.com/v2/resize:fit:592/format:webp/1*UFeVlRXco14mLNrqBjaK0w.png)

Solving for _p_, we get

![captionless image](https://miro.medium.com/v2/resize:fit:1020/format:webp/1*wLW0lin4BI-C83CsgVQp1Q.png)

We can generalise this approach by defining _u_ and _d_ as multipliers that represent the magnitudes of the upward and downward moves. The risk neutral probabilities must yield a value that is equal to the forward price for the stock factoring in interest rates. Therefore, if _r_ is interest and _t_ is the time period (in years)

![captionless image](https://miro.medium.com/v2/resize:fit:786/format:webp/1*CLj3UqfQzsqbC-fi_yUX1Q.png)
\
![captionless image](https://miro.medium.com/v2/resize:fit:1214/format:webp/1*Kn-qIu6CdKhd1ZqFjjanVA.png)

Now we can create a binomial tree with multiple periods. If we suppose that _u_ and _d_ are multiplicative inverses then an up move followed by a down move or a down move followed by an up move result in the same price. If _u_ and _d_ are kept the same throughout at every branch of the tree, then in a risk-neutral world the probability of an up movement will always be

![captionless image](https://miro.medium.com/v2/resize:fit:434/format:webp/1*xnXQF56mxCpcmIS4EJLjhw.png)

The Python implementation can be seen below

```
def create_level(previousLevel, up, down):
    newLevel = []
    for i in previousLevel:
        newLevel.append(i * down)
    newLevel.append(previousLevel[-1]*up)
    return newLevel
def create_underlying_tree(spot, up, down, numPeriods):
    underlyingTree = [[spot]]
    count = 1
    while count <= numPeriods:
        underlyingTree.append(create_level(underlyingTree[-1], up, down))
        count = count + 1
    return underlyingTree
```

With the following inputs (as seen in the example in Sheldon Natenberg’s Option Volatility and Pricing: Advanced Trading Strategies and Techniques) we obtain the tree shown below. The code for plotting the tree can be seen in the [GitHub repository.](https://github.com/JamesRv21/Binomial-Options-Pricing-Model)

```
interest = 0.04
numPeriods = 3
optionType = "Call"
timeToExpiration = 0.75
up = 1.05
down = 1/up
spot = 100
strike = 100
p = ((1 + interest*(timeToExpiration/numPeriods)) - down) / (up - down)
underlyingTree = create_underlying_tree(spot, up, down, numPeriods)
```

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*0GOt6i5yMfgI7fOHj7xEeg.png)

### Valuing an Option

We know at expiration that an option is worth exactly the intrinsic value, the maximum of [_S_-_X_, 0] for a call and the maximum of [_X_-_S_, 0] for a put, where _S_ is underlying price and _X_ is the strike price. Therefore, we can propagate back through the binomial tree, to the root node to calculate the value of the option, using the probabilities obtained from assuming a risk-neutral world.

In a one-period binomial tree, the expected value of a call is

![captionless image](https://miro.medium.com/v2/resize:fit:984/format:webp/1*SVqk0_z3zMEZnpY3K_JTNg.png)

The theoretical value of the call is the present value of the expected value

![captionless image](https://miro.medium.com/v2/resize:fit:1006/format:webp/1*3wRo1xKMSAu9Y-e-xHkRgg.png)

The implementation of this logic in Python can be seen below

```
def create_option_value_level(previousLevel, p, interest, timeToExpiration, numPeriods, optionType="Call"):
    newLevel = []
    for i in range(len(previousLevel) - 1):
        newLevel.append(((1-p) * previousLevel[i] + p * previousLevel[i+1]) / (1+(interest*timeToExpiration/numPeriods)))
    return newLevel
def create_option_value_tree(underlyingTreeLastLevel, strike, p, interest, timeToExpiration, numPeriods, optionType="Call",):
    option_value_tree = []
    if optionType == "Call":
        currentLevel = [max(0, item - strike) for item in underlyingTreeLastLevel]
    elif optionType == "Put":
        currentLevel = [max(0, strike - item) for item in underlyingTreeLastLevel]
    while(len(currentLevel) > 0):
        option_value_tree.append(currentLevel)
        currentLevel = create_option_value_level(currentLevel, p, interest, timeToExpiration, numPeriods)
    option_value_tree.reverse()
    return option_value_tree
```

Again using the same inputs as before we obtain the following binomial tree with the underlying price on top and the option’s value on the bottom at each node.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*1ak4FirWDQcRDAFw_RlnIw.png)

### The Delta (Δ)

The delta of an option is the change in the option’s value with respect to movement in the price of the underlying contract. We can express this as a fraction

![captionless image](https://miro.medium.com/v2/resize:fit:306/format:webp/1*E1qU0cu1gpmdhg-yEul59w.png)

The python implementation for this logic can be seen below

```
def create_delta_tree(underlyingTree, option_value_tree):
    delta_tree = []
    for i in range(1, len(underlyingTree)):
        priceTree = underlyingTree[i]
        optionTree = option_value_tree[i]
        currentLevel = []
        for j in range(i):
            currentLevel.append((optionTree[j+1] - optionTree[j]) / (priceTree[j+1] - priceTree[j])  * 100)
        delta_tree.append(currentLevel)
    return delta_tree
```

Once again, we will use the same inputs but this time for a **put** instead of a **call**.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*s1GaAF_ZYNNYoUgCia-TOg.png)

### American Options

If we look closely at the value of the 100 put at bottom of the penultimate level in the previous image. The theoretical value of the put is 8.31. But with an underlying price of 90.70 the put has an intrinsic value of 9.30. If the put is American, anyone holding the put will choose to exercise early. Therefore, to adjust for this exercise type, we compare the European value with the intrinsic value. If the intrinsic value is greater then the European value, we replace the value at that node with the option’s intrinsic value at each preceding node.

For an American option we calculate the intrinsic value and take the maximum if the European value and the intrinsic value. The Python implementation can be seen below.

```
def create_option_value_level_american(previousLevel, underlyingLevel, p, interest, timeToExpiration, numPeriods, optionType="Call", exerciseType = "European"):
    newLevel = []
    for i in range(len(previousLevel) - 1):
        optionValue = ((1-p) * previousLevel[i] + p * previousLevel[i+1]) / (1+(interest*timeToExpiration/numPeriods))
        if exerciseType == "American":
            if optionType == "Call":
                valueIfExercised = max(0, underlyingLevel[i] - strike) 
            elif optionType == "Put":
                valueIfExercised = max(0, strike - underlyingLevel[i])
            optionValue = max(optionValue, valueIfExercised)
        newLevel.append(optionValue)
    return newLevel
def create_option_value_tree_american(underlyingTree, strike, p, interest, timeToExpiration, numPeriods, optionType="Call", exerciseType = "European"):
    option_value_tree = []
    if optionType == "Call":
        currentLevel = [max(0, item - strike) for item in underlyingTree[-1]]
    elif optionType == "Put":
        currentLevel = [max(0, strike - item) for item in underlyingTree[-1]]
    counter = 0
    while(len(currentLevel) > 0):
        option_value_tree.append(currentLevel)
        counter += 1
        currentLevel = create_option_value_level_american(currentLevel, underlyingTree[numPeriods - counter],p, interest, timeToExpiration, numPeriods, optionType=optionType, exerciseType=exerciseType)
    option_value_tree.reverse()
    return option_value_tree
```

We see this yields a different binomial tree resulting in a different price for the option.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*nnn_10OBvuQX0bWxm4pzjw.png)

### Black-Scholes Comparison

How close are option values generated by a binomial model to those generated by the Black-Scholes model? We can only answer this question for European options as the Black-Scholes model cannot be used to evaluate American option.

We can increase the accuracy of the binomial model by increasing the number of time periods. If we build a tree with an infinite number of time periods, the binomial value will converge to the Black-Scholes value.

The accuracy of a binomial calculation can be further increased by taking the average value generated by two periods. For example, taking the average of a 9-period tree and a 10-period tree.

The figure below shows the values obtained for the binomial pricing model for different time periods. As well as the values obtained from taking the average value between consecutive time periods.

![captionless image](https://miro.medium.com/v2/resize:fit:956/format:webp/1*kgx7ZMP4xpnPXKm5SUKnRQ.png)