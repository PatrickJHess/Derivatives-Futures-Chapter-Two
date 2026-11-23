## Graph the vix against arb band

```py
def graph_uncertainty_arb(df_list, asset):
    """
    Generates a 2-tier dashboard:
    Top Row: Static side-by-side Volume and Closing Price charts.
    Bottom Row: Interactive Overview/Detail chart explicitly for Closing Price.
    """
    # 1. Prepare the combined dataset
    df_combined = pd.concat(df_list).reset_index(names='date')

    price_domain = [df_combined['close'].min() * 0.9, df_combined['close'].max() * 1.1]
    volume_domain = [df_combined['volume'].min() * 0.9, df_combined['volume'].max() * 1.1]

    # Shared base chart ensures both rows get the legend by default
    base_chart = alt.Chart(df_combined).mark_line().encode(
        color=alt.Color('symbol:N', legend=alt.Legend(title="Symbol"))
    )

    # --- TOP ROW: Static Side-by-Side ---
    volume_chart = base_chart.encode(
        x=alt.X('date:T', title='Date'),
        y=alt.Y('volume:Q', title='Volume', scale=alt.Scale(domain=volume_domain))
    ).properties(
        width=300, height=350, title=f'{asset} Trading Volume - Full Timeline'
    )

    price_chart = base_chart.encode(
        x=alt.X('date:T', title='Date'),
        y=alt.Y('close:Q', title='Price', scale=alt.Scale(domain=price_domain))
    ).properties(
        width=300, height=350, title=f'{asset} Closing Prices - Arb Effect'
    )

    top_row = volume_chart | price_chart


    # --- BOTTOM ROW: Interactive Arb Effect (Price Only) ---
    brush = alt.selection_interval(encodings=['x'])

    price_base = base_chart.encode(
        y=alt.Y('close:Q', title='Price', scale=alt.Scale(domain=price_domain))
    )

    # Upper Detailed Price Chart (Keeps the legend from base_chart)
    price_upper = price_base.encode(
        x=alt.X('date:T', scale=alt.Scale(domain=brush), title='')
    ).properties(
        width=650, height=260, title=f'Interactive Detail: {asset} Arb Effect'
    )

    # Lower Overview Price Chart (Legend explicitly hidden)
    price_lower = price_base.encode(
        x=alt.X('date:T', title='Date (Drag to Zoom)'),
        color=alt.Color('symbol:N', legend=None) 
    ).properties(
        width=650, height=150
    ).add_params(
        brush
    )

    bottom_row = price_upper & price_lower


    # --- FINAL DASHBOARD COMBINATION ---
    # We resolve the legend independently so Altair doesn't try to force 
    # the top row and bottom row to share a single, awkwardly placed legend.
    final_dashboard = (top_row & bottom_row).resolve_legend(
        color="independent"
    )

    return final_dashboard

```
