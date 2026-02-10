import streamlit as st
import pandas as pd
import plotly.graph_objects as go

# --- PAGE CONFIGURATION ---
st.set_page_config(page_title="S&P 500 Scenario Builder", layout="wide")

st.title("S&P 500 Scenario Builder (Feb 2026)")
st.markdown("""
This tool models the S&P 500 Index based on **Sector-Level Assumptions**.
Adjust the **Growth** and **P/E Multiple** for each sector to see the implied S&P 500 Price.
""")

# --- 1. BASELINE DATA (Jan 2, 2026) ---
# Hardcoded baseline to ensure stability
data = {
    'Sector': [
        'Info Tech', 'Financials', 'Health Care', 'Cons Discret', 
        'Comm Svcs', 'Industrials', 'Cons Staples', 'Energy', 
        'Utilities', 'Real Estate', 'Materials'
    ],
    'Weight': [0.315, 0.130, 0.120, 0.100, 0.090, 0.085, 0.060, 0.035, 0.025, 0.025, 0.020],
    'Current_PE': [31.0, 15.5, 19.0, 26.5, 21.5, 23.0, 20.5, 12.5, 19.0, 18.0, 20.0]
}

df_base = pd.DataFrame(data)
SP500_BASE_PRICE = 6858.47
SP500_BASE_EPS = SP500_BASE_PRICE / 22.0 # ~311.75

# --- 2. SIDEBAR INPUTS ---
st.sidebar.header("Scenario Inputs")

# Global override option
apply_global = st.sidebar.checkbox("Apply Global Assumptions?", value=False)

if apply_global:
    st.sidebar.subheader("Global Settings")
    global_growth = st.sidebar.slider("Global Earnings Growth (%)", -20, 50, 8, 1)
    global_pe_change = st.sidebar.slider("Global P/E Expansion/Contraction (%)", -50, 50, 0, 5)
    
    # Apply to all sectors
    user_growth = {sec: global_growth/100 for sec in df_base['Sector']}
    user_pe = {sec: pe * (1 + global_pe_change/100) for sec, pe in zip(df_base['Sector'], df_base['Current_PE'])}

else:
    st.sidebar.subheader("Sector Specific Settings")
    user_growth = {}
    user_pe = {}
    
    # Create an expander for each sector to save space
    for index, row in df_base.iterrows():
        with st.sidebar.expander(f"{row['Sector']} ({row['Weight']:.1%})"):
            g = st.slider(f"{row['Sector']} Growth %", -20, 50, 8, key=f"g_{index}")
            p = st.slider(f"{row['Sector']} Target P/E", 5.0, 60.0, float(row['Current_PE']), 0.5, key=f"p_{index}")
            user_growth[row['Sector']] = g / 100
            user_pe[row['Sector']] = p

# --- 3. CALCULATION ENGINE ---
# Map inputs to dataframe
df_calc = df_base.copy()
df_calc['Assumed_Growth'] = df_calc['Sector'].map(user_growth)
df_calc['Target_PE'] = df_calc['Sector'].map(user_pe)

# A. Calculate Contribution to Index EPS Growth
# Weighted Growth Contribution = Weight * Growth_Rate
# We assume sector weights roughly proxy earnings contribution weights for this simplified model
df_calc['Weighted_Growth_Contr'] = df_calc['Weight'] * df_calc['Assumed_Growth']
total_weighted_growth = df_calc['Weighted_Growth_Contr'].sum()

# New Index EPS
future_index_eps = SP500_BASE_EPS * (1 + total_weighted_growth)

# B. Calculate New Aggregate P/E (Harmonic Mean)
# Index P/E is strictly: 1 / Sum(Weight / Sector_PE)
# Note: We must re-weight based on the new market caps implied by the new P/Es, 
# but for a forward-looking "Solver", keeping the initial weights as the allocation basis is standard.
inverse_pe_sum = (df_calc['Weight'] / df_calc['Target_PE']).sum()
future_index_pe = 1 / inverse_pe_sum

# C. Solved Price
future_price = future_index_eps * future_index_pe
implied_return = (future_price / SP500_BASE_PRICE) - 1

# --- 4. MAIN DASHBOARD DISPLAY ---

# Top Level Metrics
col1, col2, col3, col4 = st.columns(4)
col1.metric("S&P 500 Target", f"{future_price:,.0f}", f"{implied_return:.1%}")
col2.metric("Implied EPS", f"${future_index_eps:.2f}", f"{(future_index_eps/SP500_BASE_EPS)-1:.1%} vs Base")
col3.metric("Implied P/E", f"{future_index_pe:.1f}x", f"{future_index_pe - 22.0:.1f}x vs Base")
col4.metric("Base Level (Jan '26)", f"{SP500_BASE_PRICE:,.0f}", "Baseline")

st.divider()

# Detailed Table
st.subheader("Sector Breakdown")
st.dataframe(df_calc[['Sector', 'Weight', 'Current_PE', 'Target_PE', 'Assumed_Growth']].style.format({
    'Weight': '{:.1%}',
    'Current_PE': '{:.1f}x',
    'Target_PE': '{:.1f}x',
    'Assumed_Growth': '{:.1%}'
}), use_container_width=True)

# Visualizations
col_charts1, col_charts2 = st.columns(2)

with col_charts1:
    st.subheader("Valuation Impact by Sector")
    # Bar chart comparing Base PE vs Target PE
    fig_pe = go.Figure(data=[
        go.Bar(name='Current P/E', x=df_calc['Sector'], y=df_calc['Current_PE']),
        go.Bar(name='Target P/E', x=df_calc['Sector'], y=df_calc['Target_PE'])
    ])
    fig_pe.update_layout(barmode='group', height=400)
    st.plotly_chart(fig_pe, use_container_width=True)

with col_charts2:
    st.subheader("Sensitivity Analysis")
    st.write("How S&P 500 Price changes if Tech P/E expands/contracts (holding other inputs constant)")
    
    # Sensitivity range for Tech PE
    tech_pe_range = range(20, 45, 1)
    prices = []
    
    # Save original tech target
    original_tech_target = user_pe['Info Tech']
    
    for p in tech_pe_range:
        # Recalculate just the PE part for the loop
        temp_pe_col = df_calc['Target_PE'].copy()
        # Find tech index
        tech_idx = df_calc.index[df_calc['Sector'] == 'Info Tech'][0]
        temp_pe_col[tech_idx] = p
        
        inv_pe = (df_calc['Weight'] / temp_pe_col).sum()
        idx_pe = 1 / inv_pe
        prices.append(future_index_eps * idx_pe)
        
    fig_sens = go.Figure(data=go.Scatter(x=list(tech_pe_range), y=prices, mode='lines+markers'))
    fig_sens.add_vline(x=original_tech_target, line_dash="dash", annotation_text="Selected Tech PE")
    fig_sens.update_layout(xaxis_title="Info Tech P/E", yaxis_title="S&P 500 Price", height=400)
    st.plotly_chart(fig_sens, use_container_width=True)
