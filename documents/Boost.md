# Introduction
Boost是基于C++标准的库，创建于1998年  
Boost源码允许任何人自由使用、修改和分发  

# Setup
WIP  

# Boost库
## Boost.Program_options
```c++
#include <boost/program_options.hpp>
namespace po = boost::program_options;
int main(int ac, char* av[]){
    // Declare the supported options.
    po::options_description desc("Allowed options");
    desc.add_options()
        ("help", "produce help message")
        ("compression", po::value<int>(), "set compression level")
    ;
    
    po::variables_map vm;
    po::store(po::parse_command_line(ac, av, desc), vm);
    po::notify(vm);    
    
    if (vm.count("help")) {
        cout << desc << "\n";
        return 1;
    }
    
    if (vm.count("compression")) {
        cout << "Compression level was set to " << vm["compression"].as<int>() << ".\n";
    } else {
        cout << "Compression level was not set.\n";
    }

}
```


## Boost.Serialization
WIP  

# Reference
www.boost.org  
https://www.boost.org/doc/libs/1_39_0/doc/html/program_options/tutorial.html#id2891824  
http://zh.highscore.de/cpp/boost/introduction.html  
