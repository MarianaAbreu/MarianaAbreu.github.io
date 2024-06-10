---
title: "ECG Signal Inversion"
published: true
---

## ECG Signal Inversion

In my research I explore **5k hours** of electrocardiography data (ECG) acquired from a hospital system. These hours are usually divided into separated files, where each file has 1 to 2 hours only. These files have two columns, one for the positive channel and one for the negative. However, the configuration may change during the acquisition, as the positive channel becomes the negative. So, I need an automatic algorithm to check the proper orientation of ECG, invert if necessary, while being computationally low.

Here is a snippet of the original signal (only the first 200 seconds are shown): 

![image info](/_data/images/ecg_raw_1000s_original.png)

The first step is to filter the entire sample (2 hours in this case) and then resample to 80Hz. This guarantees that relevant information is not loss, while reducing the memory and computational load. 

```python
import biosppy as bp
import pandas as pd
from scipy.signals import resample
# ecg filtering
ecg_filtered = bp.signals.tools.filter_signal(signal=ecg_df.values, ftype='FIR', band='bandpass', order=int(sampling_rate*1.5), frequency=[0.67, 45], sampling_rate = sampling_rate)['signal']
# ecg resampled
resampled = resample(ecg, int(len(ecg) * 80 / sampling_rate))
# resample time to match new sampling rate
resampled_time = pd.date_range(ecg_df.index[0], ecg_df.index[-1], periods=len(resampled))
# save in dataframe
ecg_df = pd.DataFrame(resampled, index=resampled_time, columns=['ECG']) 
```

![image info](/_data/images/ecg_raw_10s_filtered.png)

Now, we will look for a good enough segment of 10 seconds, evaluating whether the **kurtosis** belongs to [5, 100]. This interval includes segments with reasonable signal-to-noise ratio. Since noise interference usually affects more than one segment, if the segment is excluded, the time advances 2 minutes to find the next segment.

<img src="/_data/images/ecg_inversion_scheme.png" width="200">

The case of the segment above has a kurtosis = 7.1 after filtering, so its good enough to be used to check for signal inversion. 

1) Create two ECGs, one is the other *-1. Let's call V1 to the input and V2 to the transformed signal.


![image info](/_data/images/ecg_raw_10s_both.png)

2) Get R-peaks of both signals (r_peaks, r_peaks_i)

```python
# R-peaks locations
r_peaks = bp.signals.ecg.hamilton_segmenter(signal=good_ecg['ECG'], sampling_rate=80)['rpeaks']
# Correct R-peaks
r_peaks = bp.signals.ecg.correct_rpeaks(signal=good_ecg['ECG'], rpeaks=r_peaks, sampling_rate=80, tol=0.05)['rpeaks']
```

3) Find matching R peaks in both vectors (remove all R peaks that do not have a match)
```python
from scipy.spatial import KDTree
# Define a threshold for how close numbers need to be
threshold = 5
tree = KDTree(r_peaks_i.reshape(-1, 1))
# Query the tree for the nearest neighbor of each point in 'x' within the threshold
distances, indices = tree.query(r_peaks.reshape(-1, 1), distance_upper_bound=threshold)
# Filter 'x' and 'y' based on the query results
rp_new = r_peaks[distances != np.inf]
rpi_new = r_peaks_i[indices[distances != np.inf]]

```
4) Get the Median difference -> absolute value lower than 5? -> value positive? -> invert original signal



![image info](/_data/images/ecg_inversion_scheme2.png)



### Key Points

- Only 10 seconds of one signal is enough to check inversion.
- No need of fancy algorithms, only simple R-peak calculation
- KD tree allows to map nearest neighbours without effort
- Difference between the two series should be small and positive to raise the need for inversion.

**Happy coding! 😄**

