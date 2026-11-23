## One Y Axis

```py
def one_y_axis(x_data, y_data_list, title="", xlabel="", ylabel="",
                     series_labels=None, markers=None, colors=None,
                     figure_size=(10, 6), y_limits=None,
                     save_config=None, fill_config=None, ax=None): # <-- Added ax parameter
    """
    Plots data on a single y-axis. Can plot standalone or onto an existing subplot.

    Args:
        x_data (array-like): Data for the x-axis.
        y_data_list (list of array-like): A list of datasets for the y-axis.
        title (str, optional): The title of the graph.
        xlabel (str, optional): The label for the x-axis.
        ylabel (str, optional): The label for the y-axis.
        series_labels (list of str, optional): Identifiers for each data series. 
        markers (list of str, optional): The markers to use for each series.
        colors (list of str, optional): Colors for each series.
        figure_size (tuple, optional): The width and height of the figure in inches. Defaults to (10, 6).
        y_limits (tuple, optional): The (min, max) values for the y-axis.
        save_config (dict, optional): Config for saving the file. Keys: 'volume', 'chapter', 'file_name'.
        fill_config (dict, optional): Config for filling areas. Keys: 'Between' (list of 1 or 2 indices),
                                      'Start', 'End', 'Colors', 'Labels', 'Alpha'.
        ax (matplotlib.axes.Axes, optional): An existing axes object to plot on. If None, creates a new figure.

      Returns:
        matplotlib.axes.Axes: The axes object containing the plot.
    """
    num_series = len(y_data_list)

    # --- Smart Defaults (Frictionless Inputs) ---
    series_labels = series_labels or [f"Series {i+1}" for i in range(num_series)]
    markers = markers or [""] * num_series
    colors = colors or plt.cm.viridis_r(np.linspace(0, 1, num_series))

    # Input Validation
    if not (len(series_labels) == len(markers) == len(colors) == num_series):
        raise ValueError("Lengths of 'series_labels', 'markers', and 'colors' must match 'y_data_list'.")

    # --- Plotting Setup (Protects Global State) ---
    with plt.style.context('ggplot'):

        # Determine if we are building a new figure or using an existing subplot
        if ax is None:
            fig, current_ax = plt.subplots(figsize=figure_size)
            is_standalone = True
        else:
            current_ax = ax
            is_standalone = False

        # Use set_title so it applies correctly to subplots, not just the master figure
        if title:
            current_ax.set_title(title)

        # --- Pythonic Loop ---
        for y_data, label, marker, color in zip(y_data_list, series_labels, markers, colors):
            current_ax.plot(x_data, y_data, label=label, marker=marker, color=color)

        # --- Implement the Fill Feature ---
        if fill_config:
            indices = fill_config.get('Between', [0])
            y1 = y_data_list[indices[0]]
            y2 = y_data_list[indices[1]] if len(indices) > 1 else np.zeros_like(y1)

            start, end = fill_config.get('Start', 0), fill_config.get('End', len(x_data))

            current_ax.fill_between(
                x_data[start:end], y1[start:end], y2[start:end],
                color=fill_config.get('Colors', 'gray'),
                alpha=fill_config.get('Alpha', 0.3),
                label=fill_config.get('Labels', None)
            )

        # --- Final Touches ---
        if y_limits:
            current_ax.set_ylim(y_limits)
        current_ax.set_xlabel(xlabel)
        current_ax.set_ylabel(ylabel)
        current_ax.legend(loc='upper right')

        # --- Save & Show (Only if standalone) ---
        # We prevent plt.show() if passing an ax, so the parent script can build the full dashboard first
        if is_standalone:
            plt.tight_layout()
            if save_config:
                path = save_results(save_config=save_config)
                if path:
                    plt.savefig(path, dpi=300, bbox_inches='tight')
            plt.show()

        return current_ax

```
