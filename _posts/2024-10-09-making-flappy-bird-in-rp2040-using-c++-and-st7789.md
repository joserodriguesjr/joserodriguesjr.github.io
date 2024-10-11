---
title: Making Flappy Bird in RP2040 using C++ and ST7789
date: 2024-10-09 11:30:0 -0300
categories: [Embedded, Driver]
tags: [embedded, firmware, driver, c++, rp2040, spi]
comments: true
---

# TODO

- Multi Language
- Newsletter

- wxguy.in/posts/creating-personal-blog-powered-by-github-jekyll-and-asciidoc-for-free/

## Motivation

Last year I did a Flappy Bird clone in C for a school project, then a few months ago I rewrote it in C++ following OOP principles. Trying to learn new things, I settled for a new challenge: remaking the game (again!) for the RP2040 board using a ST7789 display.

## What are we going to do

- Setup the pico-sdk
- Write a driver
- Make the game

## Requirements

- RP2040 
- ST7789 Display
- pico-sdk
- C++
- CMake
- Docker

I'm using the [T-PicoC3](https://github.com/Xinyuan-LilyGO/T-PicoC3) that comes with an ESP32-C3, a RP2040 and a ST7789 Display connected to it.

## Setting up the environment

That's probably the easiest part. You can use my repository that have a .devcontainer with the pico-sdk ready for use. You'll just need to download Docker and VSCode, then clone the repo: [pico-sdk-devcontainer](https://github.com/joserodriguesjr/pico-sdk-devcontainer) \
Inside the repository there are more details regarding how to install it.

All the files written in this post can be found inside the repository above.

## Writing a driver for ST7789

I'll be using this [data sheet](https://www.buydisplay.com/download/ic/ST7789.pdf) for reference

### First things first

The ST7789 display uses a SPI interface


First, let's create our driver class:
```c++
struct st7789_config {
  spi_inst_t *spi;
  uint gpio_din;
  uint gpio_clk;
  int gpio_cs;
  uint gpio_dc;
  uint gpio_rst;
  uint gpio_bl;
};

class ST7789 {
public:
  ST7789(const struct st7789_config *config, uint16_t width, uint16_t height);
  ~ST7789();

  void update_display(uint16_t x, uint16_t y, const uint16_t *sprite_data, uint16_t sprite_width, uint16_t sprite_height);
  void draw_sprite(uint16_t x, uint16_t y, const uint16_t *sprite_data, uint16_t sprite_width, uint16_t sprite_height);
  void write(const uint16_t *data, size_t len);
  void set_cursor(uint16_t xs, uint16_t ys, uint16_t xe, uint16_t ye);

private:
  st7789_config config;
  bool data_mode;
  uint16_t width;
  uint16_t height;
  uint16_t back_buffer[ST77899V_WIDTH * ST77899V_HEIGHT];

  void caset(uint16_t xs, uint16_t xe);
  void raset(uint16_t ys, uint16_t ye);
  void ramwr();
  void cmd(uint8_t cmd, const uint8_t *data, size_t len);
  void cs_select(bool select);
  void dc_select(bool data_mode);
};
```
{: file="st7789.hpp" }


## Making Flappy Bird


