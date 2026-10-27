# Proposed IMU Arrow Schema

## Purpose

Represent IMU measurements in Apache Arrow using Rust while preserving
measurement time, coordinate-frame information, and uncertainty.
This prototype investigates a data format for pipes-rs; it does not
implement a sensor driver, fusion algorithm, or ROS 2 publisher.

## Layout

Each RecordBatch row represents one IMU sample.
The prototype contains three synthetic samples and 15 columns.

| Fields | Arrow type | Nullable | Meaning |
|---|---|---|---|
| measurement_time_ns | Int64 | No | Measurement time in nanoseconds in the declared clock domain |
| frame_id | Utf8 | No | Coordinate frame for the sensor measurements |
| accel_x, accel_y, accel_z | Float64 each | No | Acceleration components in m/s² |
| angular_velocity_x, angular_velocity_y, angular_velocity_z | Float64 each | No | Angular velocity components in rad/s |
| orientation_x, orientation_y, orientation_z, orientation_w | Float64 each | Yes | Unit quaternion representing sensor orientation relative to a reference frame |
| linear_acceleration_covariance | FixedSizeList<Float64, 9> | No | Acceleration covariance in (m/s²)² |
| angular_velocity_covariance | FixedSizeList<Float64, 9> | No | Angular velocity covariance in (rad/s)² |
| orientation_covariance | FixedSizeList<Float64, 9> | No | Covariance of small angular errors in rad² |

Float64 preserves the numeric precision used by ROS 2 IMU fields.
Separate component columns make individual axes easy to access.
Fixed-size lists express the required nine entries per covariance matrix.

The current list child fields allow nulls at the Arrow type level because
of the construction helper. The intended application contract requires
all nine covariance entries to be present.

## Time semantics

The demo uses these schema metadata values:

- clock_domain: synthetic_demo
- time_origin: demo start
- time_unit: nanoseconds
- timestamp_meaning: measurement time

Sample times are 0, 10000000, and 20000000 nanoseconds.
They are spaced 10 milliseconds apart.

These values are not Unix timestamps or automatically valid ROS times.
Int64 is used instead of an Arrow calendar timestamp because the demo
clock has an arbitrary origin.

All rows in this batch share the declared clock domain. Combining streams
requires an explicit clock mapping or additional per-row clock identifiers.
Measurement time must not be silently replaced with arrival time.

## Coordinate conventions

frame_id identifies the frame used for acceleration and angular velocity.
The demo uses imu_link. A frame name alone does not define its transform.

The synthetic orientation examples represent rotations about z relative
to an assumed demo reference frame. They are independently chosen examples,
not estimates calculated from the demo angular velocities.

A production schema must explicitly document the orientation reference
frame, axis conventions, transforms, and calibration.

## Covariance and missing measurements

Each covariance matrix is stored row by row:
xx, xy, xz, yx, yy, yz, zx, zy, zz.

The prototype preserves the ROS 2 IMU conventions:

- All nine zeros: covariance is unknown, not perfect certainty.
- First entry -1: the corresponding measurement is unavailable.
- Otherwise: entries describe measurement covariance.

For unavailable orientation, all four quaternion columns are null and
orientation_covariance[0] is -1. Partial quaternion nulls are invalid
under this proposed contract.

The three example rows demonstrate:

1. Orientation available, with an invented known covariance.
2. Orientation available, with unknown covariance.
3. Orientation unavailable, with the -1 sentinel.

Acceleration and angular velocity are available in all demo rows.
Their covariance values are invented examples, not calibrated estimates.

## Proposed conversion into ROS 2 sensor_msgs/Imu

For each row:

1. Validate required values, units, quaternion availability, and covariance.
2. Map measurement_time_ns into the intended ROS clock domain.
   Reject conversion if that mapping is unavailable.
3. Split the mapped integer nanoseconds into:
   sec = floor(mapped_ns / 1000000000)
   nanosec = mapped_ns - sec * 1000000000
   Check that sec fits ROS int32 and nanosec is in [0, 1000000000).
4. Copy frame_id into header.frame_id.
5. Copy acceleration components into linear_acceleration.
6. Copy angular velocity components into angular_velocity.
7. For available orientation, copy x, y, z, w without reordering.
8. For unavailable orientation, use numeric placeholder components
   (for example, all zeros) and preserve orientation_covariance[0] = -1.
   The placeholders must not be interpreted as a valid rotation.
9. Copy each covariance list into its matching nine-element ROS array
   without changing its ordering or sentinel values.

This is a conversion design, not an implemented or tested ROS adapter.
Additional Pipes metadata would need separate storage or messages because
sensor_msgs/Imu cannot carry all pipeline timing and provenance details.

## Running the example

From the repository root:

    cargo run --manifest-path imu-arrow-demo/Cargo.toml

Expected batch dimensions: 3 rows and 15 columns.

## Limitations

- Only synthetic data is used.
- No real sensor driver or ROS publisher is implemented.
- The batch is constructed in memory; no persistent Arrow file is written.
- No clock synchronization, drift correction, or reset handling exists.
- Arrival times, sequence numbers, calibration versions, and drop events
  are not yet represented.
- Missing acceleration and angular velocity need an explicit policy.
- Arrow checks structural compatibility, but the demo does not yet
  validate physical meaning, quaternion norms, or covariance consistency.
- No throughput, latency, memory-use, or zero-copy benchmarks were run.

## Findings and recommended next step

The prototype demonstrates that IMU components, covariance matrices,
missing orientation, and clock metadata can be represented in an Arrow
RecordBatch.

The recommended next implementation step is an application-level validator
with tests for known covariance, unknown covariance, unavailable orientation,
partial quaternion nulls, and invalid covariance entries.

After validation, implement a ROS conversion adapter with an explicit
clock mapping and round-trip tests. Evaluate the schema with real sensor data
before making performance claims.

## References

- ROS 2 IMU message definition:
  https://github.com/ros2/common_interfaces/blob/rolling/sensor_msgs/msg/Imu.msg
- Arrow Rust RecordBatch:
  https://arrow.apache.org/rust/arrow_array/struct.RecordBatch.html
- Arrow Rust FixedSizeListArray:
  https://arrow.apache.org/rust/arrow_array/array/struct.FixedSizeListArray.html
- Project architecture:
  architecture-and-literature-review.md
