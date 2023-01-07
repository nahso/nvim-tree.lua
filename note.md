**This file will be removed / added to doc before merge**

We should allow our user to start with zero configuration and working out of the default while giving them the option to customize slowly should they choose to do that. Ideally they won't have to do drastic rewrite of their existing customization if they want to customize further

The attribute module should communicate by notifying core attribute changes by calling `require("nvim-tree.api").set_attribute("some/path", "key", value --[[ can be of any type]], propagate_up --[[boolean, whether to copy key/value to parents]], propagate_down --[[boolean, whether to copy key/value to children]])` and `require("nvim-tree.api").clear_attribute("some/path", "key")`. Note that we'll shallow copy (i.e. pass by reference) attributes to parents and children and simply assume line builder will not modify them.

The line builder will build the line according to attributes of the node, it'll be passed as a table:
```lua
{
  full_path = "/some/full/path/f.ext",
  is_dir = false,
  link_to = nil,
  depth = 3,
  attributes = {
    git = {
      {
        value = { can = "be", arbitrary = "table" },
        origin = '/some/full/path/f.ext',
      },
    },
    diagnostics = {
      {
        value = { we = "will", pass = "this", by = "reference" },
        origin = '/some/full/path/f.ext',
      },
    },
    custom_attributes = { -- this table is guaranteed to have length of at least one.
      {
        value = function() return "can even be a function" end,
        origin = '/some/full/path/f.ext',
      },
      {
        value = 'another value propagated from /some/full',
        origin = '/some/full',
      },
    },
  },
}
```
The line builder will then return all the info necessary for nvim-tree to render the line:
```lua
{
  line = {
    { str = '│ │ │ │ ', hl = 'PaddingHighlight' },
    { str = '', hl = 'IconHighlight' },
    { str = ' ' }, -- hl can be nil
    { str = 'git.lua', hl = 'FileNameHighlight' },
  },
  sign = {
    { str = '✗', hl = 'GitDirtyHighlight' },
  },
}
```

A typical custom attribute module and line builder override will look like this:
```lua
-- create autocmd to attach attribute to path
local api = require("nvim-tree.api")
local group_id = vim.api.nvim_create_augroup('highlight_focused_file')
vim.api.nvim_create_autocmd('BufEnter', {
  group = group_id,
  callback = function(ev)
    api.set_attribute(ev.file, "focused", true, true, false)
  end
})

-- override line_builder to show the attached attribute
local nvim_tree = require "nvim-tree"
local line_builder = require "nvim-tree.line_builder"
local add_to_end = require("nvim-tree.util").add_to_end
nvim_tree.setup {
  line_builder = function (node)
    local line = line_builder(node)
    if node.attributes.focused then
      add_to_end(line, { str = "f", hl = "FocusedHighlight" }, nvim_tree.get_padding())
    end
  end
}
```
