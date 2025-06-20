// This source code is subject to the terms of the Mozilla Public License 2.0 at https://mozilla.org/MPL/2.0/
// © boitoki

//@version=6
indicator('FX Market Sessions', 'MKT Sessions', overlay=true, max_lines_count=200, max_boxes_count=200, max_labels_count=200, max_bars_back=1000, explicit_plot_zorder=true)

import boitoki/AwesomeColor/9 as ac
import boitoki/Utilities/3 as util
max_bars_back(low,500)
///////////////
// Groups
///////////////
g0                      = '// GENERAL //'
g1_01                   = '// ♯1 SESSION //'
g1_02                   = '// ♯2 SESSION //'
g1_03                   = '// ♯3 SESSION //'
g1_04                   = '// ♯4 SESSION //'
g4                      = '// BOX //'
g6                      = '// LABELS //'
g5                      = '// OPENING RANGE //'
g7                      = '// FIBONACCI LEVELS //'
g8                      = '// OPTIONS //'
g11                     = '// CANDLE //'
g10                     = '// ALERTS VISUALISED //'
g12                     = '// INFORMATION //'

///////////////
// Defined
///////////////
MAX_BARS                = 500

option_yes              = 'Yes'
option_no               = '× No'
option_extend1          = 'Yes'
option_hide             = '× Hide'
option_border_style1    = '────'
option_border_style2    = '- - - - - -'
option_border_style3    = '•••••••••'
option_chart_x          = '× No'
option_chart_1          = 'Bar color'
option_chart_2          = 'Candles'
option_opr_label1       = 'High・Low'
option_opr_label2       = 'Buy・Sell'
option_opr_label_none   = 'None'
option_candle_color1    = 'Session color'
option_candle_color2    = 'Red • Green'

fmt_price               = '{0,number,#.#####}'
fmt_pips                = '{0,number,#.#}'

icon_separator          = ' • '

color_text              = color.new(color.white, 0)
TRANSPARENT             = color.new(color.black, 100)

///////////////
// Methods
///////////////
method clear (array<string> id, int _min = 0) =>
    if array.size(id) > _min
        array.pop(id)

method clear (array<float> id, int _min = 0) =>
    if array.size(id) > _min
        array.pop(id)

///////////////
// Types
///////////////
// OHLC
type OHLC
    float open
    float high
    float low
    float close
    float hl

// Dot
type Dot
    int x
    float y
    string t = ''
    color c

method create(Dot this) =>
    label.new(this.x, this.y, this.t, style=label.style_label_center, color=TRANSPARENT, textcolor=this.c, size=size.small)

// Candle
type Candle
    box[] body
    line[] wick

method create(Candle this) =>
    this.body := array.new<box>()
    this.wick := array.new<line>()
    this

type State
    string extend_style
    bool show_fibs
    bool show_op

method is_extended (State this) =>
    this.extend_style != option_no


// OpeningRange
type OpeningRange
    float top
    float btm
    float avg
    float R1
    float R2
    float S1
    float S2
    int total_count = 0
    int reached_count = 0
    int reached_R1_count = 0
    int reached_R2_count = 0
    int reached_S1_count = 0
    int reached_S2_count = 0

// Data
type Session
    string sess
    string tz
    string name
    color colour

    float[] price_ranges
    float price_range_avg
    box[] boxes
    line[] lines
    label[] labels
    line[] oclines
    box[] ocboxes
    label[] oc_labels
    box[] opr_boxes
    line[] opr_lines
    linefill[] opr_linefills
    label[] opr_labels
    line[] fib
    int session
    bool is_extended
    bool in_session

    Candle candle
    OHLC ohlc
    State state
    OpeningRange op

	int value = na // cached session time for this bar
	int start_bar = 0 // bar_index of the session start

method create (Session this, string _extend, bool _show_fib, bool _show_op) =>
    this.boxes := array.new<box>()
    this.lines := array.new<line>()
    this.labels := array.new<label>()
    this.oclines := array.new<line>()
    this.ocboxes := array.new<box>()
    this.oc_labels := array.new<label>()
    this.opr_boxes := array.new<box>()
    this.opr_lines := array.new<line>()
    this.opr_linefills := array.new<linefill>()
    this.opr_labels := array.new<label>()
    this.fib := array.new<line>()
    this.price_ranges := array.new<float>()

    this.state := State.new(_extend, _show_fib, _show_op)
    this.candle := Candle.new().create()
    this.ohlc := OHLC.new()
    this.op := OpeningRange.new()

    this.is_extended := this.state.is_extended()

    this

method update (Session this) =>
    t = time(timeframe.period, this.sess, this.tz)
    this.value := t
    if not na(t) and na(t[1])
        this.start_bar := bar_index

method get_bars_from_start(Session this) =>
    bar_index - this.start_bar + 1

method session (Session this) =>
    this.value

method add (Session this, range_price, length = 50) =>
    this.price_ranges.unshift(range_price)
    this.price_ranges.clear(length)
    this.price_range_avg := array.avg(this.price_ranges)

///////////////
// Functions
///////////////
f_get_time_by_bar (bar_count) => timeframe.multiplier * bar_count * 60 * 1000

f_border_style (_style) =>
    switch _style
        option_border_style1 => line.style_solid
        option_border_style2 => line.style_dashed
        option_border_style3 => line.style_dotted
        => _style

f_get_label_position (_y, _side) =>
    switch _y
        'top'    => _side == 'outside' ? label.style_label_lower_left : label.style_label_upper_left
        'bottom' => _side == 'outside' ? label.style_label_upper_left : label.style_label_lower_left

f_get_started(_session) => na(_session[1]) and bool(_session)

f_get_ended(_session) => na(_session) and bool(_session[1])

f_message_limit_bars (_v) => '⚠️ This box\'s right position exceeds 500 bars(' + str.tostring(_v) + '). This box is not displayed correctly.'

f_set_line_x1 (_line, _x) =>
    if (line.get_x1(_line) != _x)
        line.set_x1(_line, _x)

f_set_line_x2 (_line, _x) =>
    if (line.get_x2(_line) != _x)
        line.set_x2(_line, _x)

f_set_box_right (_box, _x) =>
    if box.get_right(_box) != _x
        box.set_right(_box, _x)

///////////////
// Inputs
///////////////
// Timezone
i_tz                    = input.string('GMT+0', title='Timezone', options=['GMT-11', 'GMT-10', 'GMT-9', 'GMT-8', 'GMT-7', 'GMT-6', 'GMT-5', 'GMT-4', 'GMT-3', 'GMT-2', 'GMT-1', 'GMT+0', 'GMT+1', 'GMT+2', 'GMT+3', 'GMT+330', 'GMT+4', 'GMT+430', 'GMT+5', 'GMT+530', 'GMT+6', 'GMT+7', 'GMT+8', 'GMT+9', 'GMT+10', 'GMT+11', 'GMT+12'], group=g0)
i_history_period        = input.int(10, 'History', minval=0, group=g0)
i_show                  = i_history_period > 0
i_lookback              = 12 * 60



// Sessions
i_show_sess1            = input.bool(true, 'Session 1 ', group=g1_01, inline='session1_1') and i_show
i_sess1_label           = input.string('London', '', group=g1_01, inline='session1_1')
i_sess1_color           = input.color(#66D9EF, '', group=g1_01, inline='session1_1')
i_sess1_barcolor1       = input.color(#66D9EF, '•', group=g1_01, inline='session1_1')
i_sess1_barcolor2       = input.color(#66D9EF, '', group=g1_01, inline='session1_1')
i_sess1                 = input.session('0800-1700', 'Time', group=g1_01)
i_sess1_extend          = input.string(option_no, 'Extend', options=[option_no, option_extend1], group=g1_01)
i_sess1_op              = input.string(option_no, 'Opening range', group=g1_01, options=[option_yes, option_no]) != option_no and i_show
i_sess1_fib             = input.string(option_no, 'Fibonacci levels', group=g1_01, options=[option_yes, option_no]) != option_no
i_sess1_chart           = input.string(option_chart_x, 'Bar', options=[option_chart_1, option_chart_2, option_chart_x], group=g1_01)
i_sess1_barcolor        = i_sess1_chart == option_chart_1
i_sess1_plotcandle      = i_sess1_chart == option_chart_2

i_show_sess2            = input.bool(true, 'Session 2 ', group=g1_02, inline='session2_1') and i_show
i_sess2_label           = input.string('New York', '', group=g1_02, inline='session2_1')
i_sess2_color           = input.color(#FD971F, '', group=g1_02, inline='session2_1')
i_sess2_barcolor1       = input.color(#FD971F, '•', group=g1_02, inline='session2_1')
i_sess2_barcolor2       = input.color(#FD971F, '', group=g1_02, inline='session2_1')
i_sess2                 = input.session('1300-2200', 'Time', group=g1_02)
i_sess2_extend          = input.string(option_no, 'Extend', options=[option_no, option_extend1], group=g1_02)
i_sess2_op              = input.string(option_no, 'Opening range', group=g1_02, options=[option_yes, option_no]) != option_no and i_show
i_sess2_fib             = input.string(option_no, 'Fibonacci levels', group=g1_02, options=[option_yes, option_no]) != option_no
i_sess2_chart           = input.string(option_chart_x, 'Bar', options=[option_chart_1, option_chart_2, option_chart_x], group=g1_02)
i_sess2_barcolor        = i_sess2_chart == option_chart_1
i_sess2_plotcandle      = i_sess2_chart == option_chart_2

i_show_sess3            = input.bool(true, 'Session 3 ', group=g1_03, inline='session3_1') and i_show
i_sess3_label           = input.string('Tokyo', '', group=g1_03, inline='session3_1')
i_sess3_color           = input.color(#AE81FF, '', group=g1_03, inline='session3_1')
i_sess3_barcolor1       = input.color(#AE81FF, '•', group=g1_03, inline='session3_1')
i_sess3_barcolor2       = input.color(#AE81FF, '', group=g1_03, inline='session3_1')
i_sess3                 = input.session('0000-0900', 'Time', group=g1_03)
i_sess3_extend          = input.string(option_no, 'Extend', options=[option_no, option_extend1], group=g1_03)
i_sess3_op              = input.string(option_no, 'Opening range', group=g1_03, options=[option_yes, option_no]) != option_no and i_show
i_sess3_fib             = input.string(option_no, 'Fibonacci levels', group=g1_03, options=[option_yes, option_no]) != option_no
i_sess3_chart           = input.string(option_chart_x, 'Bar', options=[option_chart_1, option_chart_2, option_chart_x], group=g1_03)
i_sess3_barcolor        = i_sess3_chart == option_chart_1
i_sess3_plotcandle      = i_sess3_chart == option_chart_2

i_show_sess4            = input.bool(false, 'Session 4 ', group=g1_04, inline='session4_1') and i_show
i_sess4_label           = input.string('Sydney', '', group=g1_04, inline='session4_1')
i_sess4_color           = input.color(#FB71A3, '', group=g1_04, inline='session4_1')
i_sess4_barcolor1       = input.color(#FB71A3, '•', group=g1_04, inline='session4_1')
i_sess4_barcolor2       = input.color(#FB71A3, '', group=g1_04, inline='session4_1')
i_sess4                 = input.session('2000-0500', 'Time', group=g1_04)
i_sess4_extend          = input.string(option_no, 'Extend', options=[option_no, option_extend1], group=g1_04)
i_sess4_op              = input.string(option_no, 'Opening range', group=g1_04, options=[option_yes, option_no]) != option_no and i_show
i_sess4_fib             = input.string(option_no, 'Fibonacci levels', group=g1_04, options=[option_yes, option_no]) != option_no
i_sess4_chart           = input.string(option_chart_x, 'Bar', options=[option_chart_1, option_chart_2, option_chart_x], group=g1_04)
i_sess4_barcolor        = i_sess4_chart == option_chart_1
i_sess4_plotcandle      = i_sess4_chart == option_chart_2

// Show & Styles
i_sess_box_style        = input.string('Box', '', options=['Box', 'Background', 'Hamburger', 'Sandwich'], group=g4, inline='box_style')
i_sess_border_style     = f_border_style(input.string(option_border_style2, '', options=[option_border_style1, option_border_style2, option_border_style3], group=g4, inline='box_style'))
i_sess_border_width     = input.int(1, '', minval=0, group=g4, inline='box_style')
i_sess_box_background   = input.bool(true, 'BG color', group=g4, inline='box_style_options')
i_sess_box_dots         = input.bool(true, 'Dots', group=g4, inline='box_style_options')
i_sess_end_offset       = input.bool(true, 'Include the latest bar in the Session end', group=g4)
i_sess_bgopacity        = i_sess_box_background ? 94 : 100
i_sess_bgopacity2       = math.min(i_sess_bgopacity - 4, 100)
i_sess_border_width    := i_sess_box_style == 'Background' ? 0 : i_sess_border_width
session_end_offset      = i_sess_end_offset ? 0 : 1

// Labels
i_label_show            = input.bool(true, '', inline='label_show', group=g6) and i_show
i_label_size            = str.lower(input.string('Small', '', options=['Auto', 'Tiny', 'Small', 'Normal', 'Large', 'Huge'], inline='label_show', group=g6))
i_label_position_y      = str.lower(input.string('Top', '', options=['Top', 'Bottom'], inline='label_show', group=g6))
i_label_position_s      = str.lower(input.string('Outside', '', options=['Inside', 'Outside'], inline='label_show', group=g6))
i_label_position        = f_get_label_position(i_label_position_y, i_label_position_s)
i_label_format_name     = input.bool(true, 'Name', inline='label_format', group=g6)
i_label_format_day      = input.bool(false, 'Day', inline='label_format', group=g6)
i_label_format_price    = input.bool(false, 'Price', inline='label_format', group=g6)
i_label_format_pips     = input.bool(false, 'Pips', inline='label_format', group=g6)

// Opening range
i_o_minutes             = input.int(15, title='Periods mins', minval=1, step=1, group=g5)
i_o_minutes            := math.max(i_o_minutes, timeframe.multiplier + 1)
i_o_transp              = 65
i_o_size                = input.string(size.small, 'Label size', options=[size.auto, size.tiny, size.small, size.normal, size.large, size.huge], group=g5)
i_o_color               = input.string(option_candle_color1, 'Label color', options=[option_candle_color1, option_candle_color2], group=g5)
i_o_label_1             = input.string('High', 'Label text   ', group=g5, inline='op_label', tooltip="Type \"price\" to display the price")
i_o_label_2             = input.string('Low', '', group=g5, inline='op_label')
i_o_nowonly             = input.bool(false, 'Now only', group=g5, inline='op_options')
i_o_breakout_icon       = input.bool(false, 'Breakout flag', group=g5, inline='op_options')
i_o_target              = input.bool(false, 'Target lines', group=g5, inline='op_options')

// Candle
option_candle_body      = 'OC'
option_candle_wick      = 'OHLC'
i_show_candle           = input.bool(false, '', group=g11, inline='candle_display') //and (i_candle_border_width > 0)
i_candle                = input.string(option_candle_wick, '', options=[option_candle_wick, option_candle_body], group=g11,  inline='candle_display')
i_candle_border_width   = input.int(1, '', minval=1, group=g11,  inline='candle_display')
i_candle_color          = input.string(option_candle_color1, '', options=[option_candle_color1, option_candle_color2], group=g11, inline='candle_display')
i_candle_color_g        = input.color(#A6E22E, '', group=g11, inline='candle_display')
i_candle_color_r        = input.color(#F92672, '', group=g11, inline='candle_display')

i_candle               := i_show_candle ? i_candle : option_hide
i_show_candle_wick      = i_candle == option_candle_wick
i_sess_bgopacity2      := i_show_candle ? 100 : i_sess_bgopacity2

// Fibonacci levels
i_f_show                = input.bool(true, '', group=g7, inline='fib_display')
i_f_linestyle           = f_border_style(input.string(option_border_style1, '', options=[option_border_style1, option_border_style2, option_border_style3], group=g7, inline='fib_display'))
i_f_linewidth           = input.int(1, '', minval=1, group=g7, inline='fib_display')
i_sess1_fib            := i_f_show ? i_sess1_fib : false
i_sess2_fib            := i_f_show ? i_sess2_fib : false
i_sess3_fib            := i_f_show ? i_sess3_fib : false
i_sess4_fib            := i_f_show ? i_sess4_fib : false

// Information table
i_show_info             = input.bool(true, '', group=g12, inline='info_display')
i_info_size             = input.string(size.normal, '', options=[size.auto, size.tiny, size.small, size.normal, size.large, size.huge], group=g12, inline='info_display')
i_info_value_type       = input.string('Pips', '', options=['Price', 'Pips'], group=g12, inline='info_display')
i_info_period           = input.int(50, '', group=g12, inline='info_display')

// Alerts
i_alert1_show           = input.bool(false, 'Alerts - Sessions stard/end', group=g10)
i_alert2_show           = input.bool(false, 'Alerts - Opening range breakouts', group=g10)
i_alert3_show           = input.bool(false, 'Alerts - Price crossed session\'s High/Low after session closed', group=g10)

// ------------------------
// Drawing labels
// ------------------------
f_render_label (_show, Session data, _is_started, _color, _top, _bottom) =>
    var label my_label = na
    var int start_time = na
    session = data.session()

    v_position_y = (i_label_position_y == 'top') ? _top : _bottom
    v_label = array.new_string()
    v_chg = _top - _bottom

    if _is_started
        start_time := time

    if i_label_format_name and not na(data.name)
        array.push(v_label, data.name)

    if i_label_format_day
        array.push(v_label, util.get_day(dayofweek(start_time, i_tz)))

    if i_label_format_price
        array.push(v_label, str.format(fmt_price, v_chg))

    if i_label_format_pips
        array.push(v_label, str.format(fmt_pips, util.toPips(v_chg)))

    if _show
        if _is_started
            my_label := label.new(bar_index, v_position_y, array.join(v_label, icon_separator), textcolor=_color, color=TRANSPARENT, size=i_label_size, style=i_label_position, tooltip='Pips')

            array.push(data.labels, my_label)
            util.clear_labels(data.labels, data.is_extended ? 1 : i_history_period)

        else if bool(session)
            label.set_y(my_label, v_position_y)
            label.set_text(my_label, array.join(v_label, icon_separator))

    data

// ------------------------
// Drawing Fibonacci levels
// ------------------------
f_render_fibonacci (_show, data, _is_started, _is_ended, _x1, _x2, _color, _top, _bottom, _level, _width, _style) =>
    var line my_line = na
    session = data.session()

    if _show
        y = (_top - _bottom) * _level + _bottom

        if _is_started
            my_line := line.new(_x1, y, _x2, y, width=_width, color=color.new(_color, 30), style=_style)
            array.push(data.fib, my_line)

            if data.is_extended
                line.set_extend(my_line, extend.right)

        else if  bool(session)
            line.set_y1(my_line, y)
            line.set_y2(my_line, y)

            f_set_line_x2(my_line, _x2)
        else if _is_ended
            f_set_line_x2(my_line, _x2-session_end_offset)
    data

// ------------------------
// Drawing Opening range
// ------------------------
f_render_oprange (_show, Session data, _is_started, _is_ended, _x1, _x2, _color, _max) =>
    var int start_time  = na
    var box my_box      = na
    var line my_line1   = na
    var line my_line2   = na
    var label my_label1 = na
    var label my_label2 = na
    var bool is_opened  = false
    var float R1 = na, var float R2 = na
    var float S1 = na, var float S2 = na

    var is_reached_R1 = false
    var is_reached_R2 = false
    var is_reached_S1 = false
    var is_reached_S2 = false
    var color_gray    = ac.tradingview('gray')
    var color_orange  = ac.tradingview('orange')
    var target_line_width = 2

    session = data.session()

    top                     = ta.highest(high, _max)
    btm                     = ta.lowest(low, _max)
    is_crossover            = ta.crossover(close, box.get_top(my_box))
    is_crossunder           = ta.crossunder(close, box.get_bottom(my_box))

    var float saved_top     = na

    if _show
        if _is_started
            util.clear_boxes(data.opr_boxes, math.max(0, i_history_period - 1))
            util.clear_lines(data.opr_lines, math.max(0, (i_history_period - 1) * 6))
            util.clear_labels(data.opr_labels, math.max(0, (i_history_period - 1) * 2))

            start_time  := time
            my_box      := na
            is_opened     := true

        else if  bool(session)
            time_op = start_time + (i_o_minutes * 60 * 1000)
            time_op_delay = time_op - f_get_time_by_bar(1)

            if time <= time_op and time > time_op_delay
                top_color       = i_o_color == option_candle_color2 ? ac.tradingview('blue') : _color
                btm_color       = i_o_color == option_candle_color2 ? ac.tradingview('red')  : _color
                top_text_color  = color.from_gradient(85, 0, 100, top_color, ac.panda('white'))
                bot_text_color  = color.from_gradient(85, 0, 100, btm_color, ac.panda('white'))

                mdl             = math.avg(top, btm)
                avg1            = (data.price_range_avg * 0.786) * 0.5
                avg2            = (data.price_range_avg * 1.272) * 0.5

                saved_top := top

                R1 := math.avg(btm, mdl) + avg1
                R2 := math.avg(btm, mdl) + avg2
                S1 := math.avg(top, mdl) - avg1
                S2 := math.avg(top, mdl) - avg2

                array.push(data.opr_boxes, box.new (_x1, top, bar_index, btm, border_width=0, bgcolor=color.new(_color, i_o_transp)))
                array.push(data.opr_lines, line.new(_x1, top, _x2, top, style=line.style_dashed, color=top_color))
                array.push(data.opr_lines, line.new(_x1, btm, _x2, btm, style=line.style_dashed, color=btm_color))

                array.push(data.opr_lines, line.new(_x1, R2 , _x2, R2 , style=line.style_solid, color=i_o_target ? color.new(top_color, 40) : TRANSPARENT, width=target_line_width))
                array.push(data.opr_lines, line.new(_x1, S2 , _x2, S2 , style=line.style_solid, color=i_o_target ? color.new(btm_color, 40) : TRANSPARENT, width=target_line_width))
                array.push(data.opr_lines, line.new(_x1, R1 , _x2, R1 , style=line.style_solid, color=i_o_target ? color.new(top_color, 40) : TRANSPARENT, width=target_line_width))
                array.push(data.opr_lines, line.new(_x1, S1 , _x2, S1 , style=line.style_solid, color=i_o_target ? color.new(btm_color, 40) : TRANSPARENT, width=target_line_width))

                if i_o_target
                    array.unshift(data.opr_linefills, linefill.new(array.get(data.opr_lines, array.size(data.opr_lines)-1), array.get(data.opr_lines, array.size(data.opr_lines)-3), color.new(btm_color, 96))) // S
                    array.unshift(data.opr_linefills, linefill.new(array.get(data.opr_lines, array.size(data.opr_lines)-2), array.get(data.opr_lines, array.size(data.opr_lines)-4), color.new(top_color, 96))) // R

                t1 = i_o_label_1 == 'price' ? str.tostring(top, format.mintick) : i_o_label_1
                t2 = i_o_label_2 == 'price' ? str.tostring(btm, format.mintick) : i_o_label_2

                my_label1 := label.new(_x1-1, top, t1, yloc=yloc.price, style=label.style_label_right, color=top_color, textcolor=top_text_color, size=i_o_size, tooltip=str.tostring(top, format.mintick))
                my_label2 := label.new(_x1-1, btm, t2, yloc=yloc.price, style=label.style_label_right, color=btm_color, textcolor=bot_text_color, size=i_o_size, tooltip=str.tostring(btm, format.mintick))

                array.push(data.opr_labels, my_label1)
                array.push(data.opr_labels, my_label2)
                array.push(data.opr_boxes , my_box)

                if data.is_extended
                    box.set_extend(my_box, extend.right)

                alert('Opening range is fixed.', alert.freq_once_per_bar)
            else if is_opened
                if ta.crossover(high, R1) and (not is_reached_R1)
                    is_reached_R1 := true

                    if i_o_breakout_icon
                        label.new(bar_index, high, '×', yloc=yloc.price, style=label.style_label_center, color=TRANSPARENT, textcolor=ac.tradingview('blue'), size=size.large)

                if ta.crossover(high, R2) and (not is_reached_R2)
                    is_reached_R2 := true

                    if i_o_breakout_icon
                        label.new(bar_index, high, '×', yloc=yloc.price, style=label.style_label_center, color=TRANSPARENT, textcolor=ac.tradingview('blue'), size=size.large)

                if ta.crossunder(low, S1) and (not is_reached_S1)
                    is_reached_S1 := true

                    if i_o_breakout_icon
                        label.new(bar_index, low, '×', yloc=yloc.price, style=label.style_label_center, color=TRANSPARENT, textcolor=ac.tradingview('red'), size=size.large)

                if ta.crossunder(low, S2) and (not is_reached_S2)
                    is_reached_S2 := true

                    if i_o_breakout_icon
                        label.new(bar_index, low, '×', yloc=yloc.price, style=label.style_label_center, color=TRANSPARENT, textcolor=ac.tradingview('red'), size=size.large)
                true
            else
                if is_crossover
                    alert('Price crossed over the opening range', alert.freq_once_per_bar)

                    if i_alert2_show
                        label.new(bar_index, box.get_top(my_box), "×", color=color.blue, textcolor=ac.tradingview('blue'), style=label.style_none, size=size.large)

                if is_crossunder
                    alert('Price crossed under the opening range', alert.freq_once_per_bar)

                    if i_alert2_show
                        label.new(bar_index, box.get_bottom(my_box), "×", color=color.red, textcolor=ac.tradingview('red'), style=label.style_none, size=size.large)

        else if _is_ended

            if i_o_nowonly
                util.clear_lines(data.opr_lines, 0)
                util.clear_labels(data.opr_labels, 0)
                util.clear_boxes(data.opr_boxes, 0)

            else if array.size(data.opr_lines) > 0
                for i = 0 to 5
                    the_line = array.get(data.opr_lines, array.size(data.opr_lines) - (i + 1))
                    line.set_x2(the_line, _x2-session_end_offset)

                for i = 0 to 1
                    the_label = array.get(data.opr_labels, array.size(data.opr_labels) - (i + 1))
                    label.set_text(the_label, '●')
                    label.set_style(the_label, label.style_label_center)
                    label.set_textcolor(the_label, i_o_color == option_candle_color2 ? label.get_y(the_label) >= saved_top ? ac.tradingview('blue') : ac.tradingview('red') : _color)
                    label.set_color(the_label, TRANSPARENT)
                    label.set_size(the_label, size.tiny)
                    label.set_x(the_label,_x1)

            data.op.total_count := data.op.total_count + 1
            data.op.reached_count := (is_reached_R1 or is_reached_R2 or is_reached_S1 or is_reached_S2) ? data.op.reached_count + 1 : data.op.reached_count
            data.op.reached_R1_count := is_reached_R1 ? data.op.reached_R1_count + session_end_offset : data.op.reached_R1_count
            data.op.reached_R2_count := is_reached_R2 ? data.op.reached_R2_count + session_end_offset : data.op.reached_R2_count
            data.op.reached_S1_count := is_reached_S1 ? data.op.reached_S1_count + session_end_offset : data.op.reached_S1_count
            data.op.reached_S2_count := is_reached_S2 ? data.op.reached_S2_count + session_end_offset : data.op.reached_S2_count

            // Reset
            R1 := na, R2 := na, S1 := na, S2 := na
            is_reached_R1 := false
            is_reached_R2 := false
            is_reached_S1 := false
            is_reached_S2 := false
            is_opened := false
            saved_top := na

        is_opened
    data


// ------------------------
// Drawing candle
// ------------------------
f_render_candle (_show, Session data, _is_started, _is_ended, _color, _top, _bottom, _open, _x1, _x2) =>
    var box body = na
    var line wick1 = na
    var line wick2 = na
    session = data.session()

    border_width = i_candle_border_width
    cx = math.round(math.avg(_x2, _x1)) - math.round(border_width / 2)

    body_color = i_candle_color == option_candle_color2 ? close > _open ? i_candle_color_g : i_candle_color_r : _color
    body_color := color.new(body_color, 30)

    if _show
        if _is_started
            body    := box.new(_x1, _top, _x2, _bottom, body_color, border_width, line.style_solid, bgcolor=color.new(color.black, 100))
            wick1   := i_show_candle_wick ? line.new(cx, _top, cx, _top, color=body_color, width=border_width, style=line.style_solid) : na
            wick2   := i_show_candle_wick ? line.new(cx, _bottom, cx, _bottom, color=body_color, width=border_width, style=line.style_solid) : na

            array.push(data.candle.body, body)
            array.push(data.candle.wick, wick1)
            array.push(data.candle.wick, wick2)

            util.clear_boxes(data.candle.body, i_history_period)
            util.clear_lines(data.candle.wick, i_history_period * 2)

        else if  bool(session)
            top    = math.max(_open, close)
            bottom = math.min(_open, close)

            box.set_top(body, top)
            box.set_bottom(body, bottom)
            box.set_right(body, _x2)
            box.set_border_color(body, body_color)

            line.set_y1(wick1, _top)
            line.set_y2(wick1, top)
            f_set_line_x1(wick1, cx)
            f_set_line_x2(wick1, cx)
            line.set_color(wick1, body_color)

            line.set_y1(wick2, _bottom)
            line.set_y2(wick2, bottom)
            f_set_line_x1(wick2, cx)
            f_set_line_x2(wick2, cx)
            line.set_color(wick2, body_color)

        else if _is_ended
            box.set_right(body, bar_index-session_end_offset)

    data


// ------------------------
// Rendering limit message
// ------------------------
f_render_limitmessage (_show, Session data, _is_started, _is_ended, _x, _y, _rightbars) =>
    var label my_note = na
    session = data.session()

    if _show
        if _is_started
            if _rightbars > MAX_BARS
                my_note := label.new(_x, _y, f_message_limit_bars(_rightbars), style=label.style_label_upper_left, color=color.yellow, textalign=text.align_left, yloc=yloc.price)

        else if  bool(session)
            if _rightbars > MAX_BARS
                label.set_y(my_note, _y)
                label.set_text(my_note, f_message_limit_bars(_rightbars))
            else
                label.delete(my_note)

        else if _is_ended
            label.delete(my_note)
    data

// Rendering session
//
f_render_sessionrange (_show, Session data, _is_started, _is_ended, _color, _top, _bottom, _x1, _x2, _is_vertical = false) =>
    var line above_line = na
    var line below_line = na
    session = data.session()

    if _show
        if _is_started
            if _is_vertical
                above_line := line.new(_x1, _top, _x1, _bottom, width=i_sess_border_width, style=i_sess_border_style, color=_color)
                below_line := line.new(_x2, _top, _x2, _bottom, width=i_sess_border_width, style=i_sess_border_style, color=_color)
            else
                above_line := line.new(_x1, _top, _x2, _top, width=i_sess_border_width, style=i_sess_border_style, color=_color)
                below_line := line.new(_x1, _bottom, _x2, _bottom, width=i_sess_border_width, style=i_sess_border_style, color=_color)

            linefill.new(above_line, below_line, color.new(_color, i_sess_bgopacity))

            array.push(data.lines, above_line)
            array.push(data.lines, below_line)

            util.clear_lines(data.lines, data.is_extended ? 2 : i_history_period * 2)

            if data.is_extended
                if _is_vertical
                    line.set_extend(above_line, extend.both)
                    line.set_extend(below_line, extend.both)
                else
                    line.set_extend(above_line, extend.right)
                    line.set_extend(below_line, extend.right)

        else if bool(session)
            if _is_vertical
                line.set_y1(above_line, _bottom)
                line.set_y2(above_line, _top)
                line.set_y1(below_line, _bottom)
                line.set_y2(below_line, _top)
            else
                line.set_y1(above_line, _top)
                line.set_y2(above_line, _top)
                line.set_x2(above_line, _x2)

                line.set_y1(below_line, _bottom)
                line.set_y2(below_line, _bottom)
                line.set_x2(below_line, _x2)

        else if _is_ended
            if _is_vertical
                line.set_x1(below_line, _x2-session_end_offset)
                line.set_x2(below_line, _x2-session_end_offset)
            else
                line.set_x2(above_line, _x2-session_end_offset)
                line.set_x2(below_line, _x2-session_end_offset)

            data.add(_top[1] - _bottom[1], i_info_period)

        data

// ------------------------
// Rendering session box
// ------------------------
f_render_session (_show, Session data, _is_started, _is_ended, _color, _top, _bottom, _x1, _x2) =>
    var box my_box = na
    session = data.session()

    if _show
        if _is_started
            my_box := box.new(_x1, _top, _x2, _bottom, _color, i_sess_border_width, i_sess_border_style, bgcolor=color.new(_color, i_sess_bgopacity))
            array.push(data.boxes, my_box)

            util.clear_boxes(data.boxes, data.is_extended ? 1 : i_history_period)

            if data.is_extended
                box.set_extend(my_box, extend.right)

        else if bool(session)
            box.set_top(my_box, _top)
            box.set_bottom(my_box, _bottom)
            f_set_box_right(my_box, _x2)

        else if _is_ended
            box.set_right(my_box, bar_index-session_end_offset)
            data.add(_top[1] - _bottom[1], i_info_period)

    data

f_render_dots (_show, Session data, _is_started, _is_ended, _color, _top, _bottom, _x1, _x2) =>
    var float _open = na
    var box oc_box = na
    var line oc_line_u = na
    var line oc_line_l = na
    session = data.session()

    if _show and i_sess_box_dots
        if _is_started
            oc_line_u := line.new(_x1, open , _x2, open , color=color.new(_color, 70), style=line.style_dotted)
            oc_line_l := line.new(_x1, close, _x2, close, color=color.new(_color, 70), style=line.style_dotted)
            linefill.new(oc_line_u, oc_line_l, color.new(_color, i_sess_bgopacity2))

            array.push(data.oclines, oc_line_u)
            array.push(data.oclines, oc_line_l)

            util.clear_lines(data.oclines, data.is_extended ? 2 : i_history_period * 2)
            util.clear_labels(data.oc_labels, data.is_extended ? 2 : math.max(0, (i_history_period - 1) * 2))

            _open := open

        else if  bool(session)
            line.set_x2(oc_line_u, _x2)
            line.set_x2(oc_line_l, _x2)
            line.set_y1(oc_line_l, close)
            line.set_y2(oc_line_l, close)

        else if _is_ended
            array.push(data.oc_labels, Dot.new(_x1, _open, '●', _color).create())
            array.push(data.oc_labels, Dot.new(_x2-session_end_offset, close[1], '◉', _color).create())

            line.set_x2(oc_line_u, _x2-session_end_offset)
            line.set_x2(oc_line_l, _x2-session_end_offset)
            line.set_y1(oc_line_l, close[1])
            line.set_y2(oc_line_l, close[1])

            _open := na
    true

// ------------------------
// Drawing market
// ------------------------
f_render_main (_show, Session data, _is_started, _is_ended, _color, _top, _bottom) =>
    var x1 = 0
    var x2 = 0
    x2_offset = 1
    session = data.session()

    x0_1 = ta.valuewhen(na(session[1]) and bool(session), bar_index, 1)
    x0_2 = ta.valuewhen(na(session) and bool(session[1]), bar_index, 0)
    x0_d = math.min(math.abs(x0_2 - x0_1), MAX_BARS)
    rightbars = x0_d

    if _show
        if _is_started
            x1 := bar_index
            x2 := bar_index + x0_d

            data.ohlc.open  := open
            data.ohlc.high  := _top
            data.ohlc.low   := _bottom
            data.ohlc.hl    := _top -_bottom
            data.in_session := true

        else if  bool(session)
            true_x2         = x1 + x0_d
            rightbars      := true_x2 - bar_index
            max_bars        = bar_index + MAX_BARS
            x2             := math.min(true_x2, max_bars)

            data.ohlc.high := _top
            data.ohlc.low  := _bottom
            data.ohlc.hl   := _top - _bottom

        else if _is_ended
            data.in_session := false
            data.ohlc.open := na

    [x1, x2, data.ohlc, rightbars]


// ------------------------
// Drawing
// ------------------------
draw (_show, Session data, _extend, _show_fib, _show_op) =>
    data.update()
    session     = data.session()
    max         = math.min(i_lookback, data.get_bars_from_start())

    top         = ta.highest(high, max)
    bottom      = ta.lowest(low, max)

    col         = data.colour

    is_started  = f_get_started(session)
    is_ended    = f_get_ended(session)

    [x1, x2, ohlc, _rightbars] = f_render_main(_show, data, is_started, is_ended, col, top, bottom)

    if i_sess_box_style == 'Box' or i_sess_box_style == 'Background' or i_sess_box_style == 'Candle'
        f_render_session(_show, data, is_started, is_ended, col, top, bottom, x1, x2)
        f_render_dots(_show, data, is_started, is_ended, col, top, bottom, x1, x2)

    else if i_sess_box_style == 'Hamburger'
        f_render_sessionrange(_show, data, is_started, is_ended, col, top, bottom, x1, x2)

    else if i_sess_box_style == 'Sandwich'
        f_render_sessionrange(_show, data, is_started, is_ended, col, top, bottom, x1, x2, true)

    if i_show_candle
        f_render_candle(_show, data, is_started, is_ended, col, top, bottom, ohlc.open, x1, x2)

    if i_label_show
        f_render_label(_show, data, is_started, col, top, bottom)

    if _show_op
        f_render_oprange(_show, data, is_started, is_ended, x1, x2, col, max)

    if _show_fib
        f_render_fibonacci(_show, data, is_started, is_ended, x1, x2, col, top, bottom, 0.500, 2, line.style_solid)
        f_render_fibonacci(_show, data, is_started, is_ended, x1, x2, col, top, bottom, 0.628, i_f_linewidth, i_f_linestyle)
        f_render_fibonacci(_show, data, is_started, is_ended, x1, x2, col, top, bottom, 0.382, i_f_linewidth, i_f_linestyle)
        util.clear_lines(data.fib, i_history_period * 3)

    f_render_limitmessage(_show, data, is_started, is_ended, x1, bottom, _rightbars)

    [ bool(session), ohlc]

///////////////////
// Calculating
///////////////////
string tz = (i_tz == option_no or i_tz == '') ? na : i_tz

///////////////////
// Plotting
///////////////////
var sess1_data = Session.new(i_sess1, tz, i_sess1_label, i_sess1_color).create(i_sess1_extend, i_sess1_fib, i_sess1_op)
var sess2_data = Session.new(i_sess2, tz, i_sess2_label, i_sess2_color).create(i_sess2_extend, i_sess2_fib, i_sess2_op)
var sess3_data = Session.new(i_sess3, tz, i_sess3_label, i_sess3_color).create(i_sess3_extend, i_sess3_fib, i_sess3_op)
var sess4_data = Session.new(i_sess4, tz, i_sess4_label, i_sess4_color).create(i_sess4_extend, i_sess4_fib, i_sess4_op)

[is_sess1, sess1_ohlc] = draw(i_show_sess1, sess1_data, i_sess1_extend, i_sess1_fib, i_sess1_op)
[is_sess2, sess2_ohlc] = draw(i_show_sess2, sess2_data, i_sess2_extend, i_sess2_fib, i_sess2_op)
[is_sess3, sess3_ohlc] = draw(i_show_sess3, sess3_data, i_sess3_extend, i_sess3_fib, i_sess3_op)
[is_sess4, sess4_ohlc] = draw(i_show_sess4, sess4_data, i_sess4_extend, i_sess4_fib, i_sess4_op)

is_positive_bar         = close > open

color c_barcolor        = na
color c_plotcandle      = na

c_sess1_barcolor        = (is_sess1) ? (is_positive_bar ? i_sess1_barcolor1 : i_sess1_barcolor2) : na
c_sess2_barcolor        = (is_sess2) ? (is_positive_bar ? i_sess2_barcolor1 : i_sess2_barcolor2) : na
c_sess3_barcolor        = (is_sess3) ? (is_positive_bar ? i_sess3_barcolor1 : i_sess3_barcolor2) : na
c_sess4_barcolor        = (is_sess4) ? (is_positive_bar ? i_sess4_barcolor1 : i_sess4_barcolor2) : na

if (i_sess1_chart != option_chart_x) and is_sess1
    c_barcolor         := i_sess1_barcolor   ? c_sess1_barcolor : na
    c_plotcandle       := i_sess1_plotcandle ? c_sess1_barcolor : na

if (i_sess2_chart != option_chart_x) and is_sess2
    c_barcolor         := i_sess2_barcolor   ? c_sess2_barcolor : na
    c_plotcandle       := i_sess2_plotcandle ? c_sess2_barcolor : na

if (i_sess3_chart != option_chart_x) and is_sess3
    c_barcolor         := i_sess3_barcolor   ? c_sess3_barcolor : na
    c_plotcandle       := i_sess3_plotcandle ? c_sess3_barcolor : na

if (i_sess4_chart != option_chart_x) and is_sess4
    c_barcolor         := i_sess4_barcolor   ? c_sess4_barcolor : na
    c_plotcandle       := i_sess4_plotcandle ? c_sess4_barcolor : na

barcolor(c_barcolor)
plotcandle(open, high, low, close, color=is_positive_bar ? TRANSPARENT : c_plotcandle, bordercolor=c_plotcandle, wickcolor=c_plotcandle)


////////////////////
// Alerts
////////////////////
// Session alerts
sess1_started = is_sess1 and not is_sess1[1], sess1_ended = not is_sess1 and is_sess1[1]
sess2_started = is_sess2 and not is_sess2[1], sess2_ended = not is_sess2 and is_sess2[1]
sess3_started = is_sess3 and not is_sess3[1], sess3_ended = not is_sess3 and is_sess3[1]
sess4_started = is_sess4 and not is_sess4[1], sess4_ended = not is_sess4 and is_sess4[1]

alertcondition(sess1_started,   'Session #1 started')
alertcondition(sess1_ended,     'Session #1 ended'  )
alertcondition(sess2_started,   'Session #2 started')
alertcondition(sess2_ended,     'Session #2 ended'  )
alertcondition(sess3_started,   'Session #3 started')
alertcondition(sess3_ended,     'Session #3 ended'  )
alertcondition(sess4_started,   'Session #4 started')
alertcondition(sess4_ended,     'Session #4 ended'  )

alertcondition((not is_sess1) and ta.crossover (close, sess1_ohlc.high), 'Session #1 High crossed (after session closed)')
alertcondition((not is_sess1) and ta.crossunder(close, sess1_ohlc.low) , 'Session #1 Low crossed (after session closed)' )
alertcondition((not is_sess2) and ta.crossover (close, sess2_ohlc.high), 'Session #2 High crossed (after session closed)')
alertcondition((not is_sess2) and ta.crossunder(close, sess2_ohlc.low) , 'Session #2 Low crossed (after session closed)' )
alertcondition((not is_sess3) and ta.crossover (close, sess3_ohlc.high), 'Session #3 High crossed (after session closed)')
alertcondition((not is_sess3) and ta.crossunder(close, sess3_ohlc.low) , 'Session #3 Low crossed (after session closed)' )
alertcondition((not is_sess4) and ta.crossover (close, sess4_ohlc.high), 'Session #4 High crossed (after session closed)')
alertcondition((not is_sess4) and ta.crossunder(close, sess4_ohlc.low) , 'Session #4 Low crossed (after session closed)' )

// Alerts visualized
if i_alert1_show
    if i_show_sess1
        if sess1_started
            label.new(bar_index, close, 'Start', yloc=yloc.abovebar, color=i_sess1_color, textcolor=color_text, size=size.small, style=label.style_label_down)
        if sess1_ended
            label.new(bar_index - session_end_offset, close, 'End'  , yloc=yloc.abovebar, color=i_sess1_color, textcolor=color_text, size=size.small, style=label.style_label_down)

    if i_show_sess2
        if sess2_started
            label.new(bar_index, close, 'Start', yloc=yloc.abovebar, color=i_sess2_color, textcolor=color_text, size=size.small, style=label.style_label_down)
        if sess2_ended
            label.new(bar_index - session_end_offset, close, 'End'  , yloc=yloc.abovebar, color=i_sess2_color, textcolor=color_text, size=size.small, style=label.style_label_down)

    if i_show_sess3
        if sess3_started
            label.new(bar_index, close, 'Start', yloc=yloc.abovebar, color=i_sess3_color, textcolor=color_text, size=size.small, style=label.style_label_down)
        if sess3_ended
            label.new(bar_index - session_end_offset, close, 'End'  , yloc=yloc.abovebar, color=i_sess3_color, textcolor=color_text, size=size.small, style=label.style_label_down)

    if i_show_sess4
        if sess4_started
            label.new(bar_index, close, 'Start', yloc=yloc.abovebar, color=i_sess4_color, textcolor=color_text, size=size.small, style=label.style_label_down)
        if sess4_ended
            label.new(bar_index - session_end_offset, close, 'End'  , yloc=yloc.abovebar, color=i_sess4_color, textcolor=color_text, size=size.small, style=label.style_label_down)

plot(i_alert3_show ? sess1_ohlc.high : na, 'sess1_high', style=plot.style_linebr, color=i_sess1_color)
plot(i_alert3_show ? sess1_ohlc.low  : na, 'sess1_low' , style=plot.style_linebr, color=i_sess1_color, linewidth=2)
plot(i_alert3_show ? sess2_ohlc.high : na, 'sess2_high', style=plot.style_linebr, color=i_sess2_color)
plot(i_alert3_show ? sess2_ohlc.low  : na, 'sess2_low' , style=plot.style_linebr, color=i_sess2_color, linewidth=2)
plot(i_alert3_show ? sess3_ohlc.high : na, 'sess3_high', style=plot.style_linebr, color=i_sess3_color)
plot(i_alert3_show ? sess3_ohlc.low  : na, 'sess3_low' , style=plot.style_linebr, color=i_sess3_color, linewidth=2)
plot(i_alert3_show ? sess4_ohlc.high : na, 'sess4_high', style=plot.style_linebr, color=i_sess4_color)
plot(i_alert3_show ? sess4_ohlc.low  : na, 'sess4_low' , style=plot.style_linebr, color=i_sess4_color, linewidth=2)

plotshape(i_alert3_show and (not is_sess1) and ta.crossover (close, sess1_ohlc.high) ? sess1_ohlc.high : na, 'cross sess1_high', color=i_sess1_color, style=shape.triangleup  , location=location.absolute, size=size.tiny)
plotshape(i_alert3_show and (not is_sess1) and ta.crossunder(close, sess1_ohlc.low)  ? sess1_ohlc.low  : na, 'cross sess1_low' , color=i_sess1_color, style=shape.triangledown, location=location.absolute, size=size.tiny)
plotshape(i_alert3_show and (not is_sess2) and ta.crossover (close, sess2_ohlc.high) ? sess2_ohlc.high : na, 'cross sess2_high', color=i_sess2_color, style=shape.triangleup  , location=location.absolute, size=size.tiny)
plotshape(i_alert3_show and (not is_sess2) and ta.crossunder(close, sess2_ohlc.low)  ? sess2_ohlc.low  : na, 'cross sess2_low' , color=i_sess2_color, style=shape.triangledown, location=location.absolute, size=size.tiny)
plotshape(i_alert3_show and (not is_sess3) and ta.crossover (close, sess3_ohlc.high) ? sess3_ohlc.high : na, 'cross sess3_high', color=i_sess3_color, style=shape.triangleup  , location=location.absolute, size=size.tiny)
plotshape(i_alert3_show and (not is_sess3) and ta.crossunder(close, sess3_ohlc.low)  ? sess3_ohlc.low  : na, 'cross sess3_low' , color=i_sess3_color, style=shape.triangledown, location=location.absolute, size=size.tiny)
plotshape(i_alert3_show and (not is_sess4) and ta.crossover (close, sess4_ohlc.high) ? sess4_ohlc.high : na, 'cross sess4_high', color=i_sess4_color, style=shape.triangleup  , location=location.absolute, size=size.tiny)
plotshape(i_alert3_show and (not is_sess4) and ta.crossunder(close, sess4_ohlc.low)  ? sess4_ohlc.low  : na, 'cross sess4_low' , color=i_sess4_color, style=shape.triangledown, location=location.absolute, size=size.tiny)




//////////////
// Analysis //
var text_color      = #e1e2e4
var bgcolor         = #1a1c20
var color_green     = ac.monokai('green')
var color_orange    = ac.monokai('orange')
var color_red       = ac.monokai('red')
var color_ind_green = ac.tradingview('green')
var color_ind_gray  = color.new(ac.tradingview('gray'), 30)
var color_white     = #e1e2e4

i_show_op_analysis  = false//input.bool(true, 'Opening range table')
i_show_analysis2    = false//input.bool(false)

if i_show_info and barstate.islast
    data = array.new<Session>()

    if array.size(sess1_data.price_ranges) > 0
        data.push(sess1_data)
    if array.size(sess2_data.price_ranges) > 0
        data.push(sess2_data)
    if array.size(sess3_data.price_ranges) > 0
        data.push(sess3_data)
    if array.size(sess4_data.price_ranges) > 0
        data.push(sess4_data)

    var tbl_avg = table.new(position.top_right, 5, 5, frame_width=2, bgcolor=bgcolor, frame_color=#25272c, border_width=1, border_color=#25272c)

    table.cell(tbl_avg, 0, 0, 'Session', text_color=color.new(text_color, 60), text_size=i_info_size, text_halign=text.align_center)
    table.cell(tbl_avg, 1, 0, '',        text_color=color.new(text_color, 60), text_size=i_info_size)
    table.cell(tbl_avg, 2, 0, 'Cur',     text_color=color.new(text_color, 60), text_size=i_info_size)
    table.cell(tbl_avg, 3, 0, 'Avg('+str.tostring(i_info_period)+')', text_color=color.new(text_color, 60), text_size=i_info_size)
    table.cell(tbl_avg, 4, 0, '%',       text_color=color.new(text_color, 60), text_size=i_info_size)
    table.merge_cells(tbl_avg, 0, 0, 1, 0)

    if array.size(data) > 0
        for i = 0 to array.size(data) - 1
            my_data = array.get(data, i)
            in_session = my_data.in_session and session.ismarket
            p = my_data.ohlc.hl / my_data.price_range_avg
            col_cur = p < 0.75 ? color_green : p < 1.25 ? color_orange : color_red
            v1 = i_info_value_type == 'Price' ? str.tostring(my_data.ohlc.hl, format.mintick) : str.tostring(util.toPips(my_data.ohlc.hl), '#.#')
            v2 = i_info_value_type == 'Price' ? str.tostring(my_data.price_range_avg, format.mintick) : str.tostring(util.toPips(my_data.price_range_avg), '#.#')
            v3 = in_session ? '◉' : '●'

            color_name  = in_session ? color_white : color.new(ac.monokai('gray'), 30)
            color_ind   = in_session ? color_ind_green : color_ind_gray
            color_cur   = in_session ? col_cur : color.new(col_cur, 60)

            table.cell(tbl_avg, 0, i + 1, my_data.name, text_color=color_name, text_size=i_info_size, text_halign=text.align_left)
            table.cell(tbl_avg, 1, i + 1, v3, text_color=color_ind , text_size=size.tiny,   text_halign=text.align_center, text_valign=text.align_center)
            table.cell(tbl_avg, 2, i + 1, v1, text_color=color_cur , text_size=i_info_size, text_halign=text.align_right, text_font_family=font.family_monospace)
            table.cell(tbl_avg, 3, i + 1, v2, text_color=color_name, text_size=i_info_size, text_halign=text.align_right, text_font_family=font.family_monospace)
            table.cell(tbl_avg, 4, i + 1, str.format('{0,number,percent}', p), text_color=color_cur, text_size=i_info_size, text_halign=text.align_right, text_font_family=font.family_monospace)


if i_show_op_analysis and (barstate.islast or barstate.isconfirmed)
    data = array.new<Session>()

    if sess1_data.op.reached_count > 0
        array.unshift(data, sess1_data)
    if sess2_data.op.reached_count > 0
        array.unshift(data, sess2_data)
    if sess3_data.op.reached_count > 0
        array.unshift(data, sess3_data)
    if sess4_data.op.reached_count > 0
        array.unshift(data, sess4_data)


    var tbl_op = table.new(position.bottom_right, 7, 5, frame_width=2, bgcolor=bgcolor, frame_color=#25272c, border_width=1, border_color=#25272c)
    table.cell(tbl_op, 0, 0, 'Opening range' + (array.size(data) == 0 ? ' (No data)' : ''), text_size=i_info_size)
    table.merge_cells(tbl_op, 0, 0, 6, 0)

    if array.size(data) > 0
        table.cell(tbl_op, 0, 1, 'Session', text_size=i_info_size)
        table.cell(tbl_op, 1, 1, 'Sum', text_size=i_info_size)
        table.cell(tbl_op, 2, 1, 'R1', text_size=i_info_size)
        table.cell(tbl_op, 3, 1, 'S1', text_size=i_info_size)
        table.cell(tbl_op, 4, 1, 'R2', text_size=i_info_size)
        table.cell(tbl_op, 5, 1, 'S2', text_size=i_info_size)
        table.cell(tbl_op, 6, 1, 'Total', text_size=i_info_size)

        row_index = 2
        for i = 0 to array.size(data) -1
            my_data = array.get(data, i)
            table.cell(tbl_op, 0, row_index + i, my_data.name, text_size=i_info_size, text_color=color_white, text_halign=text.align_left)
            table.cell(tbl_op, 1, row_index + i, str.format('{0,number,percent}', my_data.op.reached_count    / my_data.op.total_count), text_font_family=font.family_monospace, text_halign=text.align_right, text_color=color_white, text_size=i_info_size)
            table.cell(tbl_op, 2, row_index + i, str.format('{0,number,percent}', my_data.op.reached_R1_count / my_data.op.total_count), text_font_family=font.family_monospace, text_halign=text.align_right, text_color=color_white, text_size=i_info_size)
            table.cell(tbl_op, 3, row_index + i, str.format('{0,number,percent}', my_data.op.reached_S1_count / my_data.op.total_count), text_font_family=font.family_monospace, text_halign=text.align_right, text_color=color_white, text_size=i_info_size)
            table.cell(tbl_op, 4, row_index + i, str.format('{0,number,percent}', my_data.op.reached_R2_count / my_data.op.total_count), text_font_family=font.family_monospace, text_halign=text.align_right, text_color=color_white, text_size=i_info_size)
            table.cell(tbl_op, 5, row_index + i, str.format('{0,number,percent}', my_data.op.reached_S2_count / my_data.op.total_count), text_font_family=font.family_monospace, text_halign=text.align_right, text_color=color_white, text_size=i_info_size)
            table.cell(tbl_op, 6, row_index + i, str.tostring(my_data.op.total_count), text_font_family=font.family_monospace, text_halign=text.align_right, text_color=color_white, text_size=i_info_size)


if i_show_analysis2 and barstate.islast
    max = 100
    g_base = ta.lowest(max)
    s3_count = 0
    s2_count = 0

    var g_lines = array.new<line>()
    x = bar_index + 20
    util.clear_lines(g_lines, 0)
    array.push(g_lines, line.new(x + 0, g_base, x + 0, g_base + sess1_data.price_range_avg, color=sess1_data.colour, width=3))
    array.push(g_lines, line.new(x + 2, g_base, x + 2, g_base + sess2_data.price_range_avg, color=sess2_data.colour, width=3))
    array.push(g_lines, line.new(x + 4, g_base, x + 4, g_base + sess3_data.price_range_avg, color=sess3_data.colour, width=3))
    array.push(g_lines, line.new(x + 6, g_base, x + 6, g_base + sess4_data.price_range_avg, color=sess4_data.colour, width=3))

    if array.size(sess1_data.price_ranges) > 0 and array.size(sess3_data.price_ranges) > 0
        for i = 0 to array.size(sess1_data.price_ranges) - 1
            s1 = array.get(sess1_data.price_ranges, i)
            s3 = array.get(sess3_data.price_ranges, i)

            if s3 / s1 > 1
                s3_count := s3_count + 1

        tbl = table.new(position.bottom_right, 3, 3, frame_width=2, bgcolor=bgcolor, frame_color=#25272c, border_width=1, border_color=#25272c)

        table.cell(tbl, 0, 0, sess3_data.name + ' > ' + sess1_data.name, text_color=#e1e2e4)
        table.cell(tbl, 1, 0, str.tostring(s3_count) + ' / ' + str.tostring(array.size(sess1_data.price_ranges)), text_color=#e1e2e4)
        table.cell(tbl, 2, 0, str.format('{0,number,percent}', s3_count / array.size(sess1_data.price_ranges) ), text_color=#e1e2e4)

        table.cell(tbl, 0, 1, syminfo.ticker + '('+ syminfo.currency +')', text_color=#e1e2e4)
        table.merge_cells(tbl, 0, 1, 2, 1)
