## La Palma weather database updater

`update-weather-database` is a command that queries the observatory environment sensors and updates the weather database for use by the [web dashboard](https://github.com/warwick-one-metre/dashboard).
It is installed as a timed system service that runs every 30 seconds.

See [Software Infrastructure](https://github.com/warwick-one-metre/docs/wiki/Software-Infrastructure) for an overview of the software architecture and instructions for developing and deploying the code.

### Software Setup

After installing `observatory-weather-database-updater`, the `update-weather-database.timer` must be enabled using:
```
sudo systemctl enable update-weather-database.timer
```

The service will automatically start on system boot, or you can start it immediately using:
```
sudo systemctl start update-weather-database.timer
```

The script requires a MySQL or MariaDB database to be hosted on the same machine.
It will connect with the user account `ops` (no password) and write to the database `ops`.

After installing `mariadb` configure the default options by running:
```
sudo mysql_secure_installation
```

If this gives an error "Cannot connect to local MySQL server through socket" then you may need to start the database service:
```
sudo systemctl start mariadb
```

Set a root password, remove anonymous accounts, disable remote root login and test database.

Log in as root (`mysql -u root -p`) and the run the following to create the ops user:

```sql
CREATE USER 'ops'@'localhost';
GRANT ALL ON ops.* TO 'ops'@'localhost';
```

The database can be created using:
```sql
SET SQL_MODE = "NO_AUTO_VALUE_ON_ZERO";
SET time_zone = "+00:00";
CREATE DATABASE IF NOT EXISTS `ops` DEFAULT CHARACTER SET utf8 COLLATE utf8_general_ci;
USE `ops`;

CREATE TABLE `weather_goto_roomalert` (
  `bin` int(10) unsigned NOT NULL,
  `date` timestamp NOT NULL DEFAULT '0000-00-00 00:00:00',
  `internal_temp` float NOT NULL,
  `internal_humidity` float NOT NULL,
  `roomalert_temp` float NOT NULL,
  `roomalert_humidity` float NOT NULL,
  `dome2_internal_temp` float NOT NULL DEFAULT -1,
  `dome2_internal_humidity` float NOT NULL DEFAULT -1,
  `last_updated` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`bin`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `weather_goto_dome2_internal` (
  `bin` int(10) unsigned NOT NULL,
  `date` timestamp NOT NULL DEFAULT '0000-00-00 00:00:00',
  `temperature` float NOT NULL,
  `temperature_valid` tinyint(1) NOT NULL,
  `relative_humidity` float NOT NULL,
  `relative_humidity_valid` tinyint(1) NOT NULL,
  `last_updated` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`bin`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `weather_goto_ups` (
  `bin` int(10) unsigned NOT NULL,
  `date` timestamp NOT NULL DEFAULT '0000-00-00 00:00:00',
  `main_ups_status` tinyint(2) NOT NULL,
  `main_ups_battery_remaining` tinyint(3) unsigned NOT NULL,
  `main_ups_load` tinyint(3) unsigned NOT NULL,
  `dome_ups_status` tinyint(2) NOT NULL,
  `dome_ups_battery_remaining` tinyint(3) unsigned NOT NULL,
  `dome_ups_load` tinyint(3) unsigned NOT NULL,
  `last_updated` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`bin`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `weather_rasa_ups` (
  `bin` int(10) unsigned NOT NULL,
  `date` timestamp NOT NULL DEFAULT '0000-00-00 00:00:00',
  `ups_status` tinyint(2) NOT NULL,
  `ups_battery_remaining` tinyint(3) unsigned NOT NULL,
  `ups_load` tinyint(3) unsigned NOT NULL,
  `last_updated` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`bin`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `weather_superwasp_ups` (
  `bin` int(10) unsigned NOT NULL,
  `date` timestamp NOT NULL DEFAULT '0000-00-00 00:00:00',
  `ups1_status` tinyint(2) NOT NULL,
  `ups1_battery_remaining` tinyint(3) unsigned NOT NULL,
  `ups1_load` tinyint(3) unsigned NOT NULL,
  `ups2_status` tinyint(2) NOT NULL,
  `ups2_battery_remaining` tinyint(3) unsigned NOT NULL,
  `ups2_load` tinyint(3) unsigned NOT NULL,
  `ups3_status` tinyint(2) NOT NULL,
  `ups3_battery_remaining` tinyint(3) unsigned NOT NULL,
  `ups3_load` tinyint(3) unsigned NOT NULL,
  `roofbattery` float NOT NULL,
  `last_updated` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`bin`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `weather_network` (
  `bin` int(10) unsigned NOT NULL,
  `date` timestamp NOT NULL DEFAULT '0000-00-00 00:00:00',
  `ngtshead` float NOT NULL DEFAULT '-1',
  `google` float NOT NULL DEFAULT '-1',
  `onemetre` float NOT NULL DEFAULT '-1',
  `goto` float NOT NULL DEFAULT '-1',
  `nites` float NOT NULL DEFAULT '-1',
  `swasp` float NOT NULL DEFAULT '-1',
  `swasp_gateway` float NOT NULL DEFAULT '-1',
  `last_updated` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`bin`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `weather_nites_roomalert` (
  `bin` int(10) unsigned NOT NULL,
  `date` timestamp NOT NULL DEFAULT '0000-00-00 00:00:00',
  `internal_temperature` float NOT NULL,
  `internal_humidity` float NOT NULL,
  `rack_temperature` float NOT NULL,
  `rack_humidity` float NOT NULL,
  `security_system` tinyint(1) NOT NULL,
  `mains_power` tinyint(1) NOT NULL,
  `last_updated` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`bin`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `weather_onemetre_raindetector` (
  `bin` int(10) unsigned NOT NULL,
  `date` timestamp NOT NULL DEFAULT '0000-00-00 00:00:00',
  `unsafe_boards` tinyint(1) NOT NULL,
  `total_boards` tinyint(1) NOT NULL,
  `port1` tinyint(1) NOT NULL,
  `last_updated` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`bin`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `weather_onemetre_roomalert` (
  `bin` int(10) unsigned NOT NULL,
  `date` timestamp NOT NULL DEFAULT '0000-00-00 00:00:00',
  `internal_temp` float NOT NULL,
  `internal_humidity` float NOT NULL,
  `roomalert_temp` float NOT NULL,
  `roomalert_humidity` float NOT NULL,
  `truss_temp` float NOT NULL,
  `hatch_closed` tinyint(1) NOT NULL,
  `trap_closed` tinyint(1) NOT NULL,
  `security_system_safe` tinyint(1) NOT NULL,
  `last_updated` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`bin`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `weather_onemetre_ups` (
  `bin` int(10) unsigned NOT NULL,
  `date` timestamp NOT NULL DEFAULT '0000-00-00 00:00:00',
  `main_ups_status` tinyint(2) NOT NULL,
  `main_ups_battery_remaining` tinyint(3) unsigned NOT NULL,
  `main_ups_load` tinyint(3) unsigned NOT NULL,
  `dome_ups_status` tinyint(2) NOT NULL,
  `dome_ups_battery_remaining` tinyint(3) unsigned NOT NULL,
  `dome_ups_load` tinyint(3) unsigned NOT NULL,
  `last_updated` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`bin`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `weather_onemetre_vaisala` (
  `bin` int(10) unsigned NOT NULL,
  `date` timestamp NOT NULL DEFAULT '0000-00-00 00:00:00',
  `temperature` float NOT NULL,
  `temperature_valid` tinyint(1) NOT NULL,
  `relative_humidity` float NOT NULL,
  `relative_humidity_valid` tinyint(1) NOT NULL,
  `wind_direction` float NOT NULL,
  `wind_direction_valid` tinyint(1) NOT NULL,
  `wind_speed` float NOT NULL,
  `wind_speed_valid` tinyint(1) NOT NULL,
  `wind_gust` float NOT NULL,
  `wind_gust_valid` tinyint(1) NOT NULL,
  `wind_lull` float NOT NULL,
  `wind_lull_valid` tinyint(1) NOT NULL,
  `pressure` float NOT NULL,
  `pressure_valid` tinyint(1) NOT NULL,
  `accumulated_rain` float NOT NULL,
  `accumulated_rain_valid` tinyint(1) NOT NULL,
  `rain_intensity` float NOT NULL,
  `rain_intensity_valid` tinyint(1) NOT NULL,
  `dew_point_delta` float NOT NULL,
  `dew_point_delta_valid` tinyint(1) NOT NULL,
  `last_updated` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`bin`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `weather_goto_vaisala` (
  `bin` int(10) unsigned NOT NULL,
  `date` timestamp NOT NULL DEFAULT '0000-00-00 00:00:00',
  `temperature` float NOT NULL,
  `temperature_valid` tinyint(1) NOT NULL,
  `relative_humidity` float NOT NULL,
  `relative_humidity_valid` tinyint(1) NOT NULL,
  `wind_direction` float NOT NULL,
  `wind_direction_valid` tinyint(1) NOT NULL,
  `wind_speed` float NOT NULL,
  `wind_speed_valid` tinyint(1) NOT NULL,
  `wind_gust` float NOT NULL,
  `wind_gust_valid` tinyint(1) NOT NULL,
  `wind_lull` float NOT NULL,
  `wind_lull_valid` tinyint(1) NOT NULL,
  `pressure` float NOT NULL,
  `pressure_valid` tinyint(1) NOT NULL,
  `accumulated_rain` float NOT NULL,
  `accumulated_rain_valid` tinyint(1) NOT NULL,
  `rain_intensity` float NOT NULL,
  `rain_intensity_valid` tinyint(1) NOT NULL,
  `dew_point_delta` float NOT NULL,
  `dew_point_delta_valid` tinyint(1) NOT NULL,
  `last_updated` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`bin`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `weather_superwasp` (
  `bin` int(10) unsigned NOT NULL,
  `date` timestamp NOT NULL DEFAULT '0000-00-00 00:00:00',
  `ext_temperature` float NOT NULL,
  `ext_humidity` float NOT NULL,
  `wind_speed` float NOT NULL,
  `wind_direction` float NOT NULL,
  `sky_temp` float NOT NULL,
  `pressure` float NOT NULL,
  `dew_point_delta` float NOT NULL,
  `last_updated` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`bin`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `weather_eumetsat_opacity` (
  `bin` int(10) unsigned NOT NULL,
  `date` timestamp NOT NULL DEFAULT '0000-00-00 00:00:00',
  `opacity` float NOT NULL,
  `last_updated` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`bin`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `weather_tng_seeing` (
  `bin` int(10) unsigned NOT NULL,
  `date` timestamp NOT NULL DEFAULT '0000-00-00 00:00:00',
  `seeing` float NOT NULL,
  `last_updated` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`bin`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `weather_robodimm_seeing` (
  `bin` int(10) unsigned NOT NULL,
  `date` timestamp NOT NULL DEFAULT '0000-00-00 00:00:00',
  `seeing` float NOT NULL,
  `last_updated` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`bin`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `weather_superwasp_roomalert` (
  `bin` int(10) unsigned NOT NULL,
  `date` timestamp NOT NULL DEFAULT '0000-00-00 00:00:00',
  `comp_room_temp` float NOT NULL,
  `comp_room_humidity` float NOT NULL,
  `cam_room_temp` float NOT NULL,
  `cam_room_humidity` float NOT NULL,
  `cam_rack_temp` float NOT NULL,
  `roomalert_temp` float NOT NULL,
  `roomalert_humidity` float NOT NULL,
  `aircon_no_airflow` tinyint(1) NOT NULL,
  `roof_closed` tinyint(1) NOT NULL,
  `roof_power` tinyint(1) NOT NULL,
  `last_updated` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`bin`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `weather_superwasp_aurora` (
  `bin` int(10) unsigned NOT NULL,
  `date` timestamp NOT NULL DEFAULT '0000-00-00 00:00:00',
  `clarity` float NOT NULL,
  `light_intensity` float NOT NULL,
  `rain_intensity` float NOT NULL,
  `last_updated` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`bin`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

```

These statements may need to be copy/pasted across a few batches.  The `mariadb` client appears to have a bug where it corrupts long paste buffers.

If you want to export / backup the database you must add the `--skip-tz-utc` argument to `mysqldump` to prevent it from breaking timestamps!
