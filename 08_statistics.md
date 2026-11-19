# 08. Descriptive statistics

## Key concepts

- The **mean** is sensitive to outliers, the **median** is robust to them. If the mean is much larger than the median, the distribution is right-skewed (there are very large values).
- The **mode** is useful for categories (the most popular payment method) but unreliable for payment amounts.
- **Variance** is the mean squared deviation from the mean; **standard deviation** is its square root, in the same units as the data.
- **n or n − 1.** The sample mean is calculated from the same data, so deviations from it are systematically smaller than deviations from the true mean. Dividing by n − 1 (Bessel's correction) makes the variance estimate unbiased. `numpy.var()` divides by n by default, while `pandas.var()` divides by n − 1.
- **Percentiles:** Q1 (25th), median (50th), Q3 (75th). The 95th and 99th are used in monitoring, for example of payment processing time.
- **IQR** = Q3 − Q1. **Tukey's rule:** a value is an outlier if it is below Q1 − 1.5·IQR or above Q3 + 1.5·IQR.

## Tasks

**1.** Payment amounts: 50, 60, 60, 70, 80, 90, 3000. Calculate the mean, median and mode. What should be shown to a manager as the "typical payment"?

Mean ≈ 487, median 70, mode 60. The typical payment is the median: the mean is pulled up by a single outlier of 3000.

**2.** The median payment is 200 and the mean is 900. What can be said about the data?

There are large outliers and the distribution is right-skewed: a few very large payments pull the mean up.

**3.** Payments: 100, 200, 300, 400. What is the median?

The number of values is even, so the median is the mean of the two middle values: (200 + 300) / 2 = 250.

**4. From SQL practice.** When comparing regions by the average interval between orders ([06, task 11](06_window_functions.md)), it's important to check the sample size: if a group has only a few customers, the average is unreliable. In such cases it's better to also look at the median and the number of observations in the group.
