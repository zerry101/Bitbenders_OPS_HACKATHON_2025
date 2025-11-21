# Bitbenders_OPS_HACKATHON_2025
A complete data platform monitoring Ontario's trade with the U.S. End-to-end pipeline covering StatCan API ingestion, Delta Lake ETL, statistical KPI generation, and GNN forecasting for trade risk.


To see the final product and its capabilities, you can **[watch the video demonstration of the BI dashboard here](https://drive.google.com/file/d/17phZYynZTFbFmmGJqs5vj87XqH_FOWxH/view?usp=drive_link)**.

<img width="2213" height="1559" alt="image" src="https://github.com/user-attachments/assets/4b880863-82f0-4c27-82b1-45fe86f04643" />


# Data Ingestion

The data ingestion process is the first stage of the pipeline. Its goal is to bring raw external data into the Lakehouse in a clean and consistent way, while keeping the original structure untouched for traceability. This aligns with the ingestion stage shown in the system level draft 

The ingestion process is handled through fetch_raw_data.ipynb, which performs the following steps.

## Connecting to the External API

The notebook sends requests to the StatCan style API endpoints to pull monthly trade records. It fetches raw import and export values, partner details, HS codes, flow types, dates, and currency attributes. No transformations happen at this stage. The goal is to receive the source as is so it can land cleanly in Bronze.

## Creating Predefined DataFrames

Before writing the data into the Lakehouse, the notebook organizes the incoming JSON response into predefined Spark DataFrames. The structure of these DataFrames matches the fields shown in the Bronze schema. This includes:

* date

* reported_key

* partner_key

* hs_codes

* flow_code

* currency_key

* trade_value
  
These DataFrames help maintain the original structure while giving us full control over how the data lands.

## Mapping Codes to Internal References

The ingestion step includes dictionary mappings that convert raw labels into standardized codes. These mappings ensure consistency across the pipeline and include items like:

* geography and country lookups

* United States state name mappings

* flow type mappings

* HS code chapter groupings

These mappings prepare the raw data so later stages can clean and normalize it without ambiguity.

## Writing Data to the Bronze Tables

Once data is fetched and structured, it is written directly into the Bronze layer inside the Lakehouse. This is done without applying business rules or transformations. The Bronze tables preserve the raw incoming values exactly as they arrived so there is always a reference point to return to if issues occur downstream.

## Monthly Sync Support

A companion notebook called **Monthly_Data_Sync.ipynb** handles scheduled updates. It fetches only new periods, keeping the Bronze tables refreshed without overwriting historical data.

Thus, these ingestion steps ensure that the Bronze layer has a reliable and consistent landing zone that accurately mirrors the source.

# Data Layers  Overview

The architecture uses three storage and transformation layers inside a Lakehouse environment.

## Bronze Layer
<img width="803" height="643" alt="Bronze_data_schema" src="https://github.com/user-attachments/assets/6df5355e-3b40-439a-a93d-b077e67d2774" />

This layer stores raw data that arrives directly from the API. Data is collected in its original structure using predefined DataFrames. No business logic is applied at this stage.

## Silver Layer
<img width="1281" height="524" alt="Silver_data_schema" src="https://github.com/user-attachments/assets/36fbfad3-1ab3-4f72-b47c-8c13b9b61018" />

This layer contains cleaned, validated, and normalized tables. The tables follow a relational structure and include all foreign keys needed for consistency. Silver represents the business friendly format of the data and creates the foundation for the Gold layer.

## Gold Layer
<img width="3070" height="1585" alt="Gold_data_schema" src="https://github.com/user-attachments/assets/4ae0953d-1486-4483-b344-1f0f5c1c2c97" />

This layer contains the dimensional and fact tables used for analytics, dashboards, and predictive features. It includes calculated metrics and is optimized for fast reporting. Gold is the layer that connects directly to Power BI.

# Bronze Layer
## Raw Source Tables

Bronze aligns with the schema shown in your Bronze screenshot. It includes:

## fact_trade
<img width="3157" height="1285" alt="image" src="https://github.com/user-attachments/assets/ffd34c5d-9a91-48e3-a288-65e871215386" />

Raw monthly trade records with fields such as:

* id

* date

* reported_key

* partner_key

* hs_codes

* flow_code

* currency_key

* trade_value

This table represents the raw ingestion of trade data from the external API. It preserves the source format and creates the foundation for later transformations.

## dim_location

Contains the mapping of location codes to human readable names and ISO2 identifiers.

## dim_hs_product

Contains product codes and descriptions based on HS classification.

## dim_flow

Contains trade flow indicators such as Import and Export.

## dim_currency

Contains currency data and ISO codes.

Bronze captures everything needed to build higher level business tables, while keeping the data untouched for traceability.

# Silver Layer
Cleaned and Normalized Schema

The Silver schema reflects the diagram you provided and follows a relational design.

## Country

* country_id

* country_name

* iso

Stores master data for countries and ensures consistent country codes across the system.

## Trade_Partner

* trade_state_id

* state_name

* country_id

Represents U S states (or other partner regions). It connects each state to a country.

## Reporter

* reporter_id

* reporter_name

* country_id

Stores reporter level information. In this project, Ontario and Canada act as the primary reporters.

## HS_Section

* section_id

* section_code

* description

Represents HS chapter level categories for products.

## TradeFlowType

* trade_flow_type_id

* trade_flow_type_name

Defines whether a record is an import or export.

## Trades
<img width="3165" height="1253" alt="image" src="https://github.com/user-attachments/assets/ba3f2b05-afbf-484f-8624-1c6e2a122d3a" />

* trades_id

* date

* trade_state_id

* reporter_id

* trade_flow_type_id

Acts as the parent table representing a single trade event for a given date, flow, reporter, and partner.

## Trade_Details
<img width="3156" height="1268" alt="image" src="https://github.com/user-attachments/assets/5fb7475f-148f-422e-abdf-5c6eaeccc690" />

* trade_details_id

* trades_id

* hs_section_id

* dollar_value

* iso

Contains the actual trade values broken down by HS section. Connects to Trades through trades_id.

Silver tables ensure the data is clean, well structured, and ready to be joined in meaningful ways. This is the layer where all business logic and cleaning steps take place.

# Gold Layer
## Dimensional Star Schema for Reporting

The Gold schema aligns with your final Gold diagram. It contains denormalized, analytics ready dimensions and fact tables used directly by Power BI.

## dim_date

* date_id

* trade_date

* year

* month

* quarter

## dim_item

* item_id

* item_name

Derived from HS sections. Represents product groups shown in the dashboard.

## dim_partner

* partner_id

* partner_name

* partner_iso

* state_province

Represents U S states and other trade partners.

## dim_reporter

* reporter_id

* reporter_name

* reporter_iso

* state_province

Represents the reporter side, which is Ontario or Canada.

## Fact_trade_import
<img width="3150" height="1269" alt="image" src="https://github.com/user-attachments/assets/52918b03-3bf8-44f9-a890-a5fa1937c187" />

* id

* trade_value

* Item_Dim_FK

* Partner_Dim_FK

* Reporter_Dim_FK

* Date_Dim_FK

## Fact_trade_export
<img width="3134" height="1226" alt="image" src="https://github.com/user-attachments/assets/aa934298-691b-42ff-a571-d24c48b2a847" />

Same structure as Fact_trade_import but for export flows.
## predicted_trade_export & ## predicted_trade_import
Follows structure as Fact_trade_import & Fact_trade_export but with predicted values for next month in advance.


Gold is the final layer that feeds the dashboard visuals with fast and well organized data.

# Predictive Analytics and Anomaly Detection
## Based on alexia-kumo-predictions.ipynb

The pipeline includes an optional predictive analytics module that uses graph based modeling. It creates a relationship graph between Ontario, U S states, HS categories, and time periods. Once the graph is created, the model identifies unusual patterns, predicts future values, and highlights anomalies.

This module is separate from the main reporting tables but uses the Silver and Gold layers as clean input sources.

# Statistical Insight Tables
## Based on Stat_script.ipynb
<img width="3169" height="1262" alt="image" src="https://github.com/user-attachments/assets/42ce86cb-47f1-4322-9c4c-ac330eb8fd70" />


This notebook computes essential statistics that help drive insights in the Power BI dashboard.

These include:

* month over month change

* year over year change

* previous month values

* previous year values

* grouped totals by state and chapter

The results are saved as small tables that the dashboard can load instantly.

# Visualization Layer
## Ontario U S Trade Monitor Dashboard

The Power BI dashboard turns all the data from the Gold and Stats layers into a clear and interactive tool.

It includes four main page types.

## Overview Page
<img width="341" height="191" alt="Dashboard (4)" src="https://github.com/user-attachments/assets/041769bf-3ec2-486b-89ed-012fb1a5cfd8" />

Shows high level trade performance for imports and exports.
Displays:

* total trade value

* year over year change

* month over month change

* multi year trend lines

* insights highlighting growth and decline

* a map of trade partners

* top five states

* top five HS product groups

## State Level Page
<img width="341" height="191" alt="Dashboard (3)" src="https://github.com/user-attachments/assets/1f5e87b4-f438-422b-ace2-4e72fe0f2760" />

Allows users to pick a U S state and explore:

* total trade with that state

* year over year and month over month changes

* trade flow breakdown

* state specific trend lines

* top product groups for that state

## Chapter Level Page
<img width="342" height="193" alt="Dashboard (2)" src="https://github.com/user-attachments/assets/37a8f3ec-ad12-4e62-9114-1299925d4b35" />

Focuses on HS chapters.
Shows:

* chapter level totals

* chapter specific YoY and MoM changes

* state breakdown for each chapter

* chapter level time series

* product level trade shares

## Anomalies and Predictive Page
<img width="344" height="195" alt="Dashboard (1)" src="https://github.com/user-attachments/assets/3daf2ec7-2137-4af0-8e8a-958193bc7f7c" />

Shows forecasted values and highlights abnormal points where trade does not follow expected patterns.
Displays:

* predicted imports and exports

* confidence bands

* anomaly markers

* MoM and YoY comparison charts

* volatility trends

This page combines historical data and predictive analysis to support forward looking decisions.
