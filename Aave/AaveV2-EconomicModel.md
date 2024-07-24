# AaveV2

## Economic Model

---

### **current liquidity rate**

The current liquidity rate refers to the annualized interest rate that depositors earn by providing liquidity. 

The formula for the current liquidity rate $LR_t$ is as follows:

$$
LR_t = \bar{R_t}U_t
$$

- $\bar{R_t}$: overall borrow rate of the asset reserve, calculated as the weighted average of  variable debt $VD_t$ and stable debt $SD_t$, the formula is as follows:

$$
\bar{R_t} = 
\begin{cases} 
0, & \text{if } D_t = 0 \\
\frac{V_{D_t}V_{R_t} + S_{D_t}\bar{S}_{R_t}}{D_t}, & \text{if } D_t > 0 
\end{cases}
$$

- $U_t$: the utilisation rate

### **cumulated liquidity index**

The cumulated liquidity index represents, starting from pool initalization and accumulating to a certain moment, how much the per unit of depositor’s liquidity has increased ?  It used to track the accumulated interest of deposited assets and updates with time and liquidity rate changes. It ensures that depositors can accurately calculate their total earnings over time as liquidity rates fluctuate.

The formula for the cumulated liquidity index  $LI_t$ is as follows:

$$
LI_t=(LR_t\Delta T_{year}+1)LI_{t-1}
$$

- $LR_t$: the liquidity rate from time $t-1$ to $t$
- $\Delta T_{year}$: the delta time (yearly period) from $t-1$ to $t$
- $LI_{t-1}$: the cumulated liquidity index at time $t-1$

The initalize the cumulated liquidity index is as follows:

$$
LI_{t0}=1*10^{27}=1 ray
$$

### reserve normalised income

The reserve normalised income represents. starting from pool initalization and accumulating to a certain moment, how much the per unit of reserve has increased? It is a normalised index reflecting the overall income of the reservers.

The formula for the reserve normalised income $NI_t$ is the same as above:

$$
NI_t=(LR_t\Delta T_{year}+1)LI_{t-1}
$$

### liquidity token

The balance of liquidity token represents the share of assets in the reserver pool. The user's balance changes with each deposits and withdrawals operation.

- Deposits
    
    when a user $x$ deposits an amount $m$ in the protocol at time $t$, his scaled balance $ScB_t(x)$ updates:
    
$$
ScB_t(x)=ScB_{t-1}(x)+\frac{m}{NI_t}
$$
    
- Withdrawals
    
    when a user $x$ withdraws an amount $m$ in the protocol at time $t$, his scaled balance $ScB_t(x)$ updates:
    
$$
ScB_t(x)=ScB_{t-1}(x)-\frac{m}{NI_t}
$$
    
- At any point in time, the aToken balance of a user $aB_t(x)$ can be written as:

$$
aB_t(x)=ScB_t(x)NI_t
$$

### total debt

The total debt is the total amount of liquidity borrowed, dept is tokenized in Aave. So the total debt can be represented by the total supply of token. The formula is as follows:

$$
D_t=SD_t+VD_t
$$

- $SD_t$: the stable rate debt token supply
- $VD_t$: the variable rate debt token supply

### cumulated variale borrow index

The cumulated variable borrow index represents, starting from pool initalization and accumulating to a certain moment, how much the per unit of variable borrow dept has increased?  The index reflects the accumulation of the borrower’ debts as time and interest rates change.

The formula for the cumulated variale borrow index $VI_t$ is as follows:

$$
VI_t=(1+\frac{R_t}{T_{year}})^{\Delta T}VI_{t-1}
$$

- $R_t$: the borrows rate at time $t$, the formula is as follows:

$$
R_t = \begin{cases}     
R^{base} + \frac{U_t}{U_{optimal}} R^{slope1} & {if } U^t < U_{optimal} \\    
R^{base} + R^{slope1} + \frac{U^t - U_{optimal}}{1 - U_{optimal}} R^{slope2} & {if } U^t \geq U_{optimal}\end{cases}
$$

- $T_{year}$: number of seconds in a year (31536000)
- $\Delta T$: the delta time from $t-1$ to $t$
- $VI_{t-1}$: the variable borrows rate at time $t-1$

### normalised cumulated variable debt

The normalised cumulated variable debt represents, starting from pool initalization and accumulating to a certain moment, how much the per unit of variable borrow dept has increased? It is a normalised index reflecting the overall variable dept.

The formula for the normalised cumulated variable dept $VN_t$ is the same as above:

$$
VN_t=(1+\frac{VR_t}{T_{year}})^{\Delta T}VI_{t-1}
$$

### variable debt token

The balance of variable debt token represents the share of all variable debt. The user's balance changes with each borrows and repays/liquidations operation.

- Borrows
    
    when a user $x$ borrows an amount of $m$ from the protocol at time $t$, his scaled balance of variable debt $ScVB_t(x)$ updates:
    
$$
ScVB_t(x)=ScVB_{t-1}(x)+\frac{m}{VN_t}
$$
    
- Repays/Liquidations
    
    when a user $x$ repays or get liquidated an amount $m$ at time $t$, his scaled balance of variable debt $ScVB_t(x)$ updates:
    
$$
ScVB_t(x)=ScVB_{t-1}(x)-\frac{m}{VN_t}
$$
    
- At any point in time, the scaled balance of user’ variable debt can be written as:

$$
VD_t(x)=ScVB_t(x)VN_t
$$

### overall stable rate

The overall stable rate represents the weighted average interest rate of all stable borrow. It usually reflects the overall cost of all stable borrow.

- Borrows
    
    when a user borrows a stable borrow of amount $SB(x)$ is issued at borrow rate $R_t$, the overall stable rate $\bar{SR_t}$ updates:
    
$$
\bar{SR_t} = \frac{SD_t  SR_{t-1} + SB(x) R_t}{SD_t + SB_{x}}
$$
    
- Repays/Liquidations
    
    when user $x$ repays or get liquidated the stable borrow above for an amount $SB(x)$ at stable rate $SR(x)$, the overall stable rate $\bar{SR_t}$ updates:
    
$$
\bar{SR_t} = \begin{cases}0, & {if } SD_t - SB(x) = 0 \\\frac{SD_t SR_{t-1} - SB(x) SR(x)}{SD_t - SB(x)}, & {if } SD_t - SB(x) > 0\end{cases}
$$
    

### stable debt token

The balance of stable debt token represents the share of all stable debt. The user's balance changes with each borrows and repays/liquidations operation.

The stable debt token balance $SD(x)$ for a user $x$ is defined as follows:

$$
SD(x) = SB(x) \left(1 + \frac{\bar{SR_t}}{T_{year}} \right) \Delta T
$$