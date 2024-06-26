# Purpose of an script

The script is designed to retrieve log files from Autonomous Mobile Robots (AMR) currently connected to a Supervisor Manager (fleet). 
These log files contain crucial information about the states of the robots during task realization process.
The script aims to convert log files into comprehensive statistics, encompassing the count of both successful and unsuccessful task executions. Additionally, statistics include errors associated with docking operations or general operational errors.

## Input

In *log_generator* user can define filter rules through var: *VB_MANAGER_TEXT_TO_FIND*.
Current rules are set up to look for those lines in log file:

    '2023-10-03 11:00:17,057 - vb_manager.am - info - PFT.9f779.1 - Action completed.'
    '2023-10-03 11:00:17,288 - vb_manager.mm - info - PFT.9f779.2 - ActionPoint(id: 'C0309') started.'
    '2023-10-03 11:00:17,288 - vb_manager.am - Action - waiting - help'
    '2023-10-03 14:44:48,989 - vb_manager.am - info - Action aborted.'
    '2023-10-03 12:02:00,397 - vb_manager.action - info - 5 CorrectModule - Position correct.'
    '2023-10-03 12:02:00,397 - vb_manager.action - error - Position incorrect because position sensor (tr) is not active'
    '2023-10-03 12:02:00,397 - vb_manager.action - Departing failed'

## Processing

### Processing pipeline

1. get log from robot -> 
2. filter for specific lines and make new file which includes only filtered lines -> 
3. create action point frame and fill it with data from filtered file, accordingly to actual rules ->
4. include action point data into statistics

### Rules of processing single AP

Rules for creating Action points from filtered log file are located in:
*log_comparer._create_single_ap_dataframe_template*

    'Robot': [cls.robot_id],
    'TIME_START': ["Initial"],
    'TIME_END': ["Initial"],
    'AP_NAME': ["None"],
    'ACTION_ERROR(waiting, aborted)': [0],
    'DOCK_TRY(Position incorrect)': [0],
    'DOCK_CORRECT(Position correct)': [0],
    'UNDOCK_INCORRECT(Departing failed)': [0]

Data between brackets () defines which words in line from filtered log file corresponds to frame updates in AP dataframe.
Example:

    DOCK_TRY(Position incorrect)

if found lines "Position incorrect" in filtered log file line, value of DOCK_TRY frame in AP dataframe should be incremented.

    "Position incorrect" 			  -> DOCK_TRY +1
    "waiting", "aborted" 			  -> ACTION ERROR +1
    "Position correct" 			      -> DOCK_CORRECT +1
    "Departing failed" 			      -> UNDOCK_INCORRECT +1

### Processing intermediate data 

After filtered log file was processed, dataframe consisting of all found AP is generated:

    Robot	TIME_START	TIME_END	AP_NAME	ACTION_ERROR(waiting, aborted)	DOCK_TRY(Position incorrect)	DOCK_CORRECT(Position correct)	UNDOCK_INCORRECT(Departing failed)
    robotone	2023-10-03 11:00:17	2023-10-03 11:00:17	C0309	1	3	1	0
    robotone	2023-10-03 11:00:17	2023-10-03 11:00:17	R2D2	1	3	0	0
    robotone	2023-10-03 11:00:17	2023-10-03 11:00:17	SLAVE0	0	2	1	0
    robotone	2023-10-03 11:00:17	2023-10-03 11:00:17	IPS0	0	2	1	0
    robotone	2023-10-03 11:00:17	2023-10-03 11:00:17	C3P0	0	2	1	0
    robottwo	2023-10-03 11:00:17	2023-10-03 11:00:17	C0309	2	3	0	1
    robottwo	2023-10-03 11:00:17	2023-10-03 11:00:17	C0309	2	3	0	1
    robottwo	2023-10-03 11:00:17	2023-10-03 11:00:17	IPS0	2	3	0	1
    robottwo	2023-10-03 11:00:17	2023-10-03 11:00:17	C0309	0	2	1	0
    robottwo	2023-10-03 11:00:17	2023-10-03 11:00:17	IPS0	2	2	0	0
    robottwo	2023-10-03 11:00:17	2023-10-03 11:00:17	ATAT0	0	2	1	0

### Processing statistics data

Every single AP is processed and added into statistics dataframe.
Rules for creating statistics dataframe from filtered log file location:
*log_data_interpretation._create_ap_df_frame()*

    'AP_NAME()': name,
    'SUCC_RATE()': succ_rate,
    'SUCCESS()': success,
    'FAILURE()': failure,
    'ACTION_ERROR(ACTION_ERROR, DOCK_CORRECT)': action_error,
    'DOCK_ERROR(DOCK_TRY, UNDOCK_INCORRECT)': dock_error

Data between brackets () defines which frames from AP dataframe should be processed, when updating
statistics dataframe with single AP data.

Rules for processing and calculating statistics based on single AP data are located in:
*log_data_interpretation._fill_inserted_ap_frame()*

Current rules are set to:

    found ACTION_ERROR 	  	                            -> ACTION ERROR + 1
    DOCK_TRY>=3 or UNDOCK_INCORRECT>0 	                -> DOCK_ERROR + 1
    found ACTION_ERROR or found DOCK_ERROR              -> FAILURE() + 1
    not found ACTION_ERROR or not found DOCK_ERROR      -> SUCCES() + 1

## Output

Example statistics dataframe:

    AP_NAME()	SUCC_RATE()	SUCCESS()	FAILURE()	ACTION_ERROR(ACTION_ERROR, DOCK_CORRECT)	DOCK_ERROR(DOCK_TRY, UNDOCK_INCORRECT)
    C0309	43.0	3	4	4	4
    R2D2	0.0	    0	1	1	1
    SLAVE0	100.0	1	0	0	0
    IPS0	60.0	3	2	2	1
    C3P0	40.0	2	3	3	1
    ATAT0	67.0	2	1	1	0
    XWING0	50.0	2	2	2	1
    TIE0	100.0	1	0	0	0

### Links

More detailed chart structure:

[links](https://fotografiaartyzmi01.atlassian.net/wiki/spaces/BLS/pages/14745602/Struktura+Bsst-log-sim-test)
